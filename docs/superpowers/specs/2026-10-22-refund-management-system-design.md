# Design: NovaCart AI-Powered Refund Management System — Demo Prototype v1

Date: 2026-10-22 | Status: Approved design (demo prototype) | Source: brainstorming session against the BRD draft v0.1 and NovaCart policy documents

## 1. Purpose & scope

Build a runnable **demo prototype** of the NovaCart refund lifecycle — not a production system. It must convincingly demonstrate, to a mock customer, that the BRD's Must controls are genuinely implemented:

- **BR-04** — no refund exceeds the remaining eligible amount; concurrent requests can never duplicate execution
- **BR-05** — high-risk/ambiguous claims route to human review; a risk score is not proof of fraud
- **BR-06** — idempotent refund execution; success/fail/timeout/retry handled safely
- **BR-08** — customer statuses never promise unverified settlement dates
- **BR-09** — every decision, rule version, and financial event is auditable end-to-end

Deterministic rules are authoritative; AI risk assessment is advisory only and never computes amounts.

### v1 scenario scope

1. **Failed payment** — customer debited, order not confirmed. Exercises the RBI turnaround clock (card/UPI T+5, wallet T+1) and refund-to-original-payment-method.
2. **Missing grocery item** — item-level partial refund on a multi-item quick-commerce order (the BRD's five-minute scenario: substitution attempt, prior claim, pending payment event). Exercises proportional discount/tax allocation.

Out of scope for v1: marketplace seller settlements/TCS, returns/pickup/inspection flows, damaged-goods evidence classification, `ai_mode=claude` (a stub exists; API key via env only, never committed).

## 2. Stack & scaffolding

- **Python 3.14 + FastAPI**, managed by **uv** (reproducible `uv.lock`, `src` layout, hatchling backend, `[project.scripts]` entry points).
- **SQLite** via SQLAlchemy 2.0, WAL mode — zero infra; `data/demo.sqlite3` (gitignored).
- **Temporal** for orchestration. `temporalio` pinned **≥1.34.0** (verified: Python 3.14 support + 3.14 sandbox fixes landed by 1.34; ABI3 wheels). Dev server: `temporal server start-dev --ui --db-filename ./data/temporal-dev.db` — Web UI at :8233 doubles as a second audit view. Fallback if cp314 resolution fails: pin project to Python 3.13.
- **Jinja2 + htmx** server-rendered UI; no frontend build step.
- **ruff** (format + lint), **pytest + pytest-asyncio**.
- Config: pydantic-settings `.env`; `.env.example` is flags/ports only, no secrets.

```
src/novacart_refunds/
├── config.py                  # db url, temporal host/namespace/ports, review_timeout_minutes,
│                              # auto_approve_cap_paise, payment_mock_default_mode, recon interval, ai_mode
├── kpi.py                     # STP %, turnaround, duplicate-attempts-prevented, recon match %
├── db/                        # base.py (WAL/foreign_keys pragmas), models.py, seed.py
├── domain/
│   ├── enums.py, money.py     # money = signed BigInteger paise
│   ├── audit.py               # hash-chained audit writer
│   ├── decision.py            # deterministic auto-process gate
│   ├── policy/                # registry.py (fail-closed versioned loader), failed_payment.py (fp_v1),
│   │                          # missing_item.py (mi_v1), allocation.py (pure proportional math)
│   └── risk/                  # interface.py (RiskAssessor Protocol), demo.py (canned, seeded, deterministic),
│                              # claude_stub.py (out of v1; env key only, raises without one)
├── adapters/                  # registry.py (single wiring point), orders.py, payments.py,
│                              # settlement.py, notification.py
├── temporal/                  # shared.py (sandbox-safe dataclasses, NO SQLAlchemy imports), activities.py,
│                              # refund_workflow.py, reconciliation_workflow.py, schedules.py, worker.py
├── api/                       # main.py (lifespan: init_db + Temporal client), deps.py, errors.py, routes/
└── ui/                        # templates/ + static/styles.css
tests/
scripts/                       # smoke.py, dev.ps1
```

## 3. Temporal topology

**One worker process, two task queues:**

- `refund-lifecycle-q` — per-claim workflows + their activities (latency-sensitive).
- `reconciliation-q` — scheduled reconciliation workflow + settlement activities (batch; isolatable so demo can pause it).

**Workflow per claim:** id `refund-{claim_id}`, started with `WorkflowIDReusePolicy.REJECT_DUPLICATE`. FastAPI catches already-started → `200 {"status": "already_active", ...}` so the UI can show the duplicate-resubmission banner. DB unique constraints and the reservation insert are backup layers 2 and 3.

**The 9 BRD stages map to the workflow:**

| BRD stage | Where | Activity |
|---|---|---|
| 1. Intake | pre-workflow (API creates claim row + evidence) | — |
| 2. Identity & verification | activity `verify_identity_transaction` | customer/order/payment/fulfilment via `OrderSystemAdapter` |
| 3. Policy & eligibility | activity `evaluate_policy` | versioned rule set from `policy_rule_versions` |
| 4. Duplicate & financial checks | activity `reserve_refundable` | prior claims/executed refunds check; INSERT reservation |
| 5. AI risk assessment | activity `assess_risk` | `RiskAssessor.assess()` → advisory row |
| 6. Decision & approval | activity `record_decision` + durable wait | deterministic `decide()` gate |
| 7. Execution | activity `execute_refund` | BR-06 mechanics (below) |
| 8. Settlement & reconciliation | separate **Temporal Schedule** | cross-claim matching |
| 9. Notification & closure | activity `notify_and_close` | BR-08-safe wording |

Every activity is idempotently keyed on claim_id.

**Human review = durable wait via Workflow Update.** The workflow calls `wait_condition(review_decision is not None, timeout=review_timeout)`. The reviewer UI's Approve/Reject/Override posts reach the live workflow as a Workflow Update `review-decision`, executed with `handle.execute_update(...)` for a synchronous ack. On timeout: exception queue entry (`review_timeout`) + SLA reminder notification — the workflow **keeps waiting**, so late review still works. This demonstrates durability rather than polling.

**Execution (BR-06) sequence inside `execute_refund`:**

1. INSERT `refund_instructions` row (state `created`, unique `idempotency_key = "refund-{claim_id}-instr-{n}"`) **before** the provider call — local source of truth.
2. Call `PaymentProviderAdapter.execute_refund` with that key.
3. Map outcomes: accepted → `accepted` + ledger debit + reservation consumed; rejected → `failed` + reservation released + exception queue; timeout/ambiguous → `timeout_ambiguous` then provider `pre_check(idempotency_key)` ("did you actually take it?") before any retry; replaying a used key returns `duplicate_detected` (provider-side idempotency simulated by the fake).

**Reconciliation = Temporal Schedule** (`reconciliation-every-2m` in dev): ingest settlement batch → match `provider_ref`/`idempotency_key` against instructions + ledger → deltas/unmatched into `exception_queue` + audit. Schedule preferred over in-claim polling because it spans all claims, each run is independently visible in the Temporal Web UI, and pause/resume is free (demo control). On-demand trigger: `POST /api/demo/reconciliation/run`.

**Windows/ops notes:** three consoles — (1) `temporal server start-dev --ui --port 7233 --db-filename ./data/temporal-dev.db`, (2) `uv run refund-worker`, (3) `uv run uvicorn novacart_refunds.api.main:app --reload --port 8000`. Never co-run the worker inside uvicorn `--reload` (double registration). First dev-server run downloads a few hundred MB.

## 4. Data model (SQLite; money = signed BigInteger paise)

| Table | Purpose / notable columns |
|---|---|
| `customers` | id, name, email, phone (masked in UI), kyc_status |
| `orders` | channel (quick_commerce/marketplace), status (incl. `failed_payment`), money breakdown (subtotal/discount/tax/delivery_fee/total), payment_txn_id |
| `order_items` | sku, qty_ordered/qty_delivered, unit/line money values |
| `payment_transactions` | method (upi/card/wallet), **`token_ref` only — no PAN column exists by design** (regulatory anchor in the schema), status (`captured|failed|settled|refunded_partial|refunded_full`), settlement_expected_days |
| `refund_claims` | claim_no (`NC-2026-0001`), scenario, item_ids JSON, evidence JSON, status/workflow stage, eligible_amount, risk_band, decided_by, decision_reason, `rule_versions` JSON, `tat_deadline` |
| `refund_reservations` | claim_id FK, amount, state (reserved/consumed/released); **partial unique index on claim_id WHERE state='reserved'** (SQLite supports) — hard BR-04 guard |
| `refund_instructions` | amount, destination_token_ref, **unique idempotency_key**, indexed provider_ref, state (created/submitted/accepted/rejected/timeout_ambiguous/duplicate_detected/failed/credited), provider_response JSON, attempt_count |
| `ledger_entries` | signed amount; entry_type (reservation/refund_debit/penalty/adjustment/note) |
| `settlement_records` | provider_name, settlement_batch_id, provider_ref, instruction_id (nullable — unmatched is the demo case), amount, settled_at |
| `exception_queue` | kind (settlement_mismatch/unmatched_settlement/timeout_ambiguous/duplicate_attempt/review_timeout/provider_error/manual), ref, payload, state (open/resolved), resolution |
| `risk_assessments` | assessor, score 0–100, band (low/medium/high), signals JSON, recommended_action, **mandatory advisory note** ("Advisory only — not evidence of fraud") |
| `policy_rule_versions` | rule_set (fp/mi), semver version (unique per set), owner ("Finance & Legal — pending approval (mock)"), JSON definition (TAT days, caps, allocation basis), sha256 |
| `audit_events` | ts, claim (nullable), stage 1–9, actor (system/customer/reviewer:X/demo_ai), event_type, payload JSON, rule_version, **prev_hash/self_hash — cheap tamper-evident chain** |
| `customer_notifications` | claim, safe_status (BR-08 wording preserved verbatim), body, sent_at |

Pragmas: `foreign_keys=ON`, `journal_mode=WAL`, `busy_timeout=5000`. Sessions run short `session.begin()` transactions so the reservation insert is atomic across the API and worker processes. Hot indexes: claims(status), claims(order_id), audit_events(claim_id, ts), exception_queue(state).

## 5. Mock adapters (Protocols; concretions wired only in `adapters/registry.py`)

```python
class OrderSystemAdapter(Protocol):
    def get_order(self, order_id: str) -> OrderDetails: ...          # raises NotFoundError
    def get_fulfilment(self, order_id: str) -> FulfilmentInfo: ...   # delivered qtys, sensor flags
    def get_customer(self, customer_id: str) -> CustomerDetails: ...

class PaymentProviderAdapter(Protocol):
    def pre_check(self, idempotency_key: str) -> RefundResult | None: ...
    def execute_refund(self, request: RefundRequest) -> RefundResult: ...

class FailureMode(StrEnum):
    instant_accept; timeout_then_accept; reject; duplicate_callback; always_timeout; unavailable

class SettlementReportAdapter(Protocol):
    def fetch_batch(self, after_ts: datetime) -> list[SettlementRecord]: ...

class CustomerNotifier(Protocol):
    def send_claim_update(self, customer, claim, safe_status: str, fields: dict) -> None: ...

class RiskAssessor(Protocol):
    def assess(self, claim_ctx: ClaimContext) -> RiskAdvice: ...      # advisory only
```

The fake provider keeps an in-memory `dict[idempotency_key, outcome]` so replays yield `duplicate_detected`; its `FailureMode` is runtime-switchable via `POST /api/demo/payment-mode` — this is what makes timeout-then-accept demoable live. Activities receive adapters through the registry; tests inject stubs; no activity imports a mock directly.

## 6. Deterministic rules engine

Pure functions, zero AI input, **versioned in `policy_rule_versions`**, loaded **fail-closed** (unknown version ⇒ exception queue, never a silent default).

```python
def evaluate_policy(claim: ClaimContext, rules: RuleSet) -> PolicyOutcome
# eligible, breakdown: list[AmountLine], eligible_amount_paise, rule_version, notes, tat_deadline
```

- **`fp_v1` (failed payment):** eligible iff order.status == `failed_payment` AND txn.status == `captured` AND `token_ref` matches the original payment method (the regulatory anchor lives in *code*). Amount = full captured amount. `tat_deadline = captured_at + (T+5 card/upi, T+1 wallet)` from rule JSON. ₹100/day penalty computed as a flaggable placeholder line.
- **`mi_v1` (missing item):** missing qty from fulfilment adapter; per-line base = `unit_price_paise × qty_missing`; `allocation.distribute_order_level()` allocates order-level discount and tax **proportionally by line share**; delivery fee fully refundable when any grocery item is missing. Rounding to the paise, remainder to the highest-value line (property-tested). Rule JSON flagged `_placeholder: "awaiting Finance/Legal approval"`, surfaced in UI/audit.
- **`decide(policy_outcome, risk_advice, settings)`:** auto-approve only if eligible AND band ≠ high AND amount ≤ `auto_approve_cap_paise`; else reviewer queue or reject. Records the exact rule version + risk band behind every decision — reproducible.

## 7. API surface

JSON error envelope (`errors.py`): NotFoundError→404, validation→422, `WorkflowAlreadyStarted`→**200 `already_active`** (+current status), TemporalUnavailable→503. Demo auth: `X-Reviewer-Id` header.

- `POST /api/claims` — submit claim, start workflow
- `GET /api/claims/{id}` — status, BR-08-safe customer status
- `GET /api/orders/{id}` — items/prior-claim hints for the submission form
- `GET /api/reviewer/queue`; `POST /api/reviewer/claims/{id}/approve|reject|override` — via Workflow Update
- `GET /api/claims/{id}/audit`; `GET /api/exceptions` + `POST /api/exceptions/{id}/resolve`; `GET /api/kpi`
- Demo: `POST /api/demo/seed` (idempotent), `POST /api/demo/payment-mode`, `POST /api/demo/risk-seed` (force high band), `POST /api/demo/reconciliation/run`, `GET /api/demo/state`
- `GET /healthz`, `GET /healthz/temporal`

Resilience: Temporal client calls go through a `temporal_proxy` helper — on RPC failure, read-only routes serve SQLite state and the UI shows an "orchestrator offline" banner; submissions return 503 with retry hint.

## 8. UI (Jinja2 + htmx, one stylesheet, no build step)

| Page | Contents |
|---|---|
| `/` claim submission | scenario toggle, seeded-order picker, item checkboxes/qty, evidence, submit → claim page |
| `/claims/{id}` | live timeline: 2s htmx poll showing stage, rule version, amount breakdown, advisory banner, instruction state, notifications; polling self-stops on terminal status |
| `/reviewer` | queue with expandable evidence + rule breakdown + risk advisory; Approve/Reject/Override `hx-post` → row swap + "decision recorded" ack; SLA timer |
| `/claims/{id}/audit` | hash-chained events (actor, stage, rule version) + deep link into Temporal Web UI (:8233) |
| `/exceptions` | open exceptions incl. mismatch/timeout/duplicate/review-timeout, resolve action |
| `/kpi` | STP %, turnaround, duplicate attempts prevented, reconciliation match % |

## 9. Testing strategy (`uv run pytest` — green without a Temporal server)

- **Pure domain:** allocation invariant (sum of allocated lines equals the pool — no paise lost/gained, remainder policy property-tested), fp TAT boundaries (card/UPI T+5 vs wallet T+1), decision gate, fail-closed registry.
- **BR-04, three layers:** (a) DB — two sessions insert a reservation for the same claim, exactly one succeeds; (b) workflow — `WorkflowEnvironment.start_time_skipping()` + two `start_workflow` calls with the same id → exactly one `execute_refund` execution; (c) provider — same idempotency key replayed → `duplicate_detected`, exactly one ledger debit.
- **BR-06 failure matrix:** unit-test `execute_refund` per `FailureMode` — `timeout_then_accept` resolved via `pre_check`; `reject`; `always_timeout` → retry cap → exception queue with reservation intact.
- **API smoke:** FastAPI TestClient with Temporal client monkeypatched. Optional live-integration test behind a `--integration` flag.

## 10. Demo script (seeded world)

Seed (idempotent): 3 customers; the 5-item quick-commerce grocery order with "Basmati Rice 5kg" missing; a UPI failed-payment order (₹1,999 captured, order not confirmed); a wallet failed-payment order (T+1 contrast); a **prior claim by the same customer** (already refunded, provider_ref known); a settlement batch with one deliberate mismatch.

| Beat | Shows |
|---|---|
| A — failed payment STP | stages 2–6 auto-approve (fp_v1, T+5 deadline displayed, "advisory: low risk 0.12"), execute → close; audit hash chain + Temporal UI |
| A2 — timeout-then-accept | mock mode flipped live; `timeout_ambiguous` → `pre_check` → `duplicate_detected` → credited, **exactly one** ledger debit |
| A3 — forced high risk | `risk-seed` → claim routes to reviewer despite policy eligibility; AI advisory, deterministic decision |
| B1 — duplicate resubmission | prior claim resubmitted → REJECT_DUPLICATE + "already active" banner, KPI +1, **zero paise moved** |
| B2 — real missing-item claim | item-level breakdown with proportional discount/tax allocation + flagged delivery-fee placeholder; reviewer approves; settlement; mismatch lands in `/exceptions`; resolve on screen; close on `/kpi` |

## 11. Build order (milestones → implementation plan)

M0 Scaffold → M1 Contracts & data → M2 Domain (pure; no Temporal) → M3 Temporal layer → M4 API → M5 UI → M6 Tests & polish → M7 Demo hardening + CLAUDE.md/README finalization.

**M0 early-verify:** `uv add temporalio` resolves a cp314 wheel on Windows; else pin Python 3.13 before proceeding.

## 12. Risks

1. **temporalio ↔ Python 3.14** — supported ≥1.34 (verified); live M0 check; fallback Python 3.13.
2. **SQLAlchemy/pydantic-settings on 3.14** — import smoke in M0; pin SQLAlchemy ≥2.0.44 if needed.
3. **Workflow sandbox determinism** — no `random`/`uuid4`/wall-clock in workflow code; use `workflow.uuid7()`/`workflow.now()` or derive ids/timestamps in activities; `shared.py` stays free of SQLAlchemy imports.
4. **Windows** — dev-server download on first run; `--db-filename` so history survives restarts; never run the worker inside uvicorn `--reload`; PowerShell env quoting in scripts.
5. **SQLite two-process concurrency** — WAL + `busy_timeout` + short transactions; reservation insert in one transaction.
6. **Update semantics** — confirm `execute_update` returns the handler result synchronously when building M4 reviewer UI.
7. **Money rounding** — explicit remainder policy, property-tested.
8. **Scope guard** — marketplace settlement/TCS stays out of v1; `settlement_records.provider_name` kept generic for later extension.