# Design: NovaCart AI-Powered Refund Management System — Demo Prototype v2

Date: 2026-10-22 | Revised: 2026-10-23 | Status: Approved design v2 (demo prototype) | Source: brainstorming session against the BRD draft v0.1 and NovaCart policy documents, revised after an independent system design review

**v2 changelog (from the 2026-10-23 design review):** P0-1 cross-claim reservation invariant with serialized balance computation (§3.1, §4, §6.3); P0-2 defined handling of unknown provider status / `pending_reconciliation` (§3.2); P0-3 crash-safe execution sequence, ledger idempotency, restart recovery (§3.3); P0-4 per-component partial-refund accounting and allocation basis (§6.3); outbox dispatch, reviewer auth, audit verification, review-decision races, policy lifecycle, closure conditions, request idempotency, observability (§3.4, §4, §7, §12.9); expanded P0/P1 test matrix (§9).

## 1. Purpose & scope

Build a runnable **demo prototype** of the NovaCart refund lifecycle — not a production system. It must credibly demonstrate, to a mock customer, that the BRD's Must controls are genuinely implemented — and, per the design review, that the guarantees hold **across concurrency, provider ambiguity and process failure**, not only in the happy path:

- **BR-04** — no refund (across *all* claims, not one) exceeds the remaining approved refundable balance; concurrent requests can never duplicate execution
- **BR-05** — high-risk/ambiguous claims route to human review; a risk score is not proof of fraud; review actions are authenticated and race-safe
- **BR-06** — idempotent refund execution; success/fail/timeout/**unknown**/retry handled safely, with crash recovery between every persistence boundary
- **BR-08** — customer statuses never promise unverified settlement dates; each status has defined entry/exit conditions
- **BR-09** — every decision, rule version, and financial event is auditable end-to-end, with a verifiable audit chain

Deterministic rules are authoritative; AI risk assessment is advisory only and never computes amounts.

### v1 scenario scope

1. **Failed payment** — customer debited, order not confirmed. Exercises the RBI turnaround clock (card/UPI T+5, wallet T+1) and refund-to-original-payment-method.
2. **Missing grocery item** — item-level partial refund on a multi-item quick-commerce order (the BRD's five-minute scenario: substitution attempt, prior claim, pending payment event). Exercises proportional discount/tax allocation and **per-component caps across repeated partial refunds**.

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
├── db/                        # base.py (WAL/foreign_keys/busy_timeout pragmas), models.py, seed.py
├── domain/
│   ├── enums.py, money.py     # money = signed BigInteger paise
│   ├── audit.py               # hash-chained audit writer + verify() + canonical serialization
│   ├── decision.py            # deterministic auto-process gate
│   ├── balances.py            # serialized refundable-balance computation & reservation guard (§3.1)
│   ├── allocation.py          # pure proportional math + rounding/remainder policy (§6.3)
│   ├── policy/                # registry.py (fail-closed versioned loader), failed_payment.py (fp_v1),
│   │                          # missing_item.py (mi_v1)
│   └── risk/                  # interface.py (RiskAssessor Protocol), demo.py (canned, seeded, deterministic),
│                              # claude_stub.py (out of v1; env key only, raises without one)
├── adapters/                  # registry.py (single wiring point), orders.py, payments.py,
│                              # settlement.py, notification.py
├── temporal/                  # shared.py (sandbox-safe dataclasses, NO SQLAlchemy imports), activities.py,
│                              # refund_workflow.py, reconciliation_workflow.py, schedules.py, worker.py,
│                              # dispatcher.py (outbox drainer)
├── api/                       # main.py (lifespan: init_db + Temporal client + outbox drainer), deps.py,
│                              # errors.py, auth.py (demo auth adapter), routes/
└── ui/                        # templates/ + static/styles.css
tests/
scripts/                       # smoke.py, dev.ps1
```

## 3. Temporal topology & financial guarantees

**One worker process, two task queues:**

- `refund-lifecycle-q` — per-claim workflows + their activities (latency-sensitive).
- `reconciliation-q` — scheduled reconciliation workflow + settlement activities (batch; isolatable so demo can pause it).

**Workflow per claim:** id `refund-{claim_id}`, started with `WorkflowIDReusePolicy.REJECT_DUPLICATE`. FastAPI catches already-started → `200 {"status": "already_active", "claim_id": ..., "claim_status": ...}` — the schema is fixed and includes the existing claim's identifier and live status so the UI can show the duplicate-resubmission banner. DB unique constraints and the serialized reservation guard are backup layers 2 and 3.

**Temporal is not the financial source of truth.** It coordinates durable execution; SQLite and the ledger enforce financial invariants; the provider is authoritative for its own payment outcome. Workflow deduplication complements — never replaces — database constraints and provider-side idempotency.

### 3.1 Cross-claim refundable-balance invariant (BR-04, review P0-1)

The financial scope for all v1 scenarios is the **payment transaction** (`payment_txn_id`), subdivided into **item scopes** (per `order_item_id`) and **order-level components** (discount, tax, delivery fee). The invariant is enforced by **computing** the remaining balance inside a serialized transaction at reservation time — not by maintaining a cached balance row that could diverge from the ledger:

> **Invariant B1:** For every payment transaction P and every scope key S (any item i under P, and P's order-level components C), `Σ(active reservations + refunds credited/debited) allocated to (P, S) ≤ approved_refundable_value(P, S)` — the total is also always ≤ P's captured amount.

**Mechanism — `reserve_refundable` runs inside one `BEGIN IMMEDIATE` transaction** (SQLite serializes writers; concurrent sessions block on the write lock rather than racing):

1. `BEGIN IMMEDIATE`
2. Re-check prior claims / executed refunds for the payment & item scopes (stage-4 duplicate check).
3. `SELECT` and sum all reservations in states `reserved`/`consumed`/`credited` and all ledger `refund_debit` entries overlapping the requested scopes.
4. If any scope would exceed its approved value ⇒ `INSERT` fails deliberately: raise `OverlappingRefundError` → claim rejected with reason `duplicate_or_exhausted` (BR-04 KPI counter +1), no reservation created.
5. `INSERT refund_reservations` row carrying its **scope breakdown JSON** (`[{item_id, qty, components: {base, discount, tax, delivery_fee}}…, {order_level: …}]`) with state `reserved`, and the audit event.
6. `COMMIT`

This one transaction closes all three duplicate scenarios separately: (a) repeated claim against same order (claim-level), (b) two *different* claims racing for overlapping balance (this §3.1 guard), (c) repeated provider callbacks/executions (§3.2, §3.3). Acceptance test: **two distinct claims concurrently reserving overlapping balance — the union of accepted reservations never exceeds the balance** (§9).

Reservations are **partial-friendly**: one claim creates exactly one reservation that carries its multi-item scope breakdown; the scope model (not one-reservation-per-item) is what stays flexible. Multiple instruction attempts against one claim draw from the *same* reservation (§3.3).

### 3.2 Refund execution (BR-06, review P0-2)

`execute_refund` sequence, with each local change in its own short SQLite transaction:

1. `INSERT refund_instructions` (state `created`, `idempotency_key = "refund-{claim_id}-instr-{n}"`, exact serialized request payload) and **commit** — durable before any network call.
2. `UPDATE` state `submitted`, **commit**.
3. Call `PaymentProviderAdapter.execute_refund`.
4. **Outcome mapping — a missing result is never treated as a failure:**
   - `accepted` → state `accepted`, commit → ledger `refund_debit` (idempotent insert, §3.3) → reservation state `consumed` → `credited` after callback confirm.
   - `rejected` → state `failed`, reservation `released`, exception queue `provider_error`.
   - `timeout / connection error` → state `timeout_ambiguous`, then `pre_check(idempotency_key)`:
     - returns a verdict → map as above (same-key query only; **never a new key**).
     - returns nothing/unknown → state **`pending_reconciliation`**: reservation **stays held** (never released), exception queue `timeout_ambiguous`, and the scheduled reconciliation workflow re-queries the provider by the same idempotency key. Retry of the *same* key is permitted only up to `max_attempts`, documented as relying on the provider's idempotency contract; attempts counter on the row.
5. Provider callbacks (acceptance/credit events) are **deduplicated on the instruction's idempotency key**: a callback for an already-terminal instruction is recorded + audited, with zero ledger effect. Late success events reconcile against `pending_reconciliation` instructions exactly once.

**Provider acceptance ≠ settlement.** `accepted`/`credited` is operational; "settled" is asserted only by reconciliation matches against settlement records (§3.5). Customer messaging keys off this distinction (BR-08).

### 3.3 Crash-safe recovery (review P0-3)

- **No SQLite transaction is ever held open across a provider/network call.** Every local transition is its own committed transaction.
- **Ledger idempotency:** unique index on `ledger_entries (instruction_id, entry_type)` (where `instruction_id` is not null). A replayed `execute_refund` whose provider call already succeeded inserts at most one `refund_debit` — the second insert violates the constraint and is treated as a no-op success. Ledger postings are thus uniquely attributable per instruction and cannot duplicate under replay.
- **Ordering ambiguity resolved by idempotent replays:** a crash between "provider accepted" and "ledger posted" replays the activity; `execute_refund` is idempotent (instruction row + provider key + ledger unique index all keyed to the instruction). The reverse crash (ledger committed, process died mid-way) resumes at the next step, never re-posts (unique index).
- **Restart recovery:** workflow history (Temporal) replays and resumes at the durable point; the activity contract above means replay converges to one correct financial outcome. The worker also runs a startup sweep: any instruction in `created`/`submitted` with a stale `updated_at` (crash before outcome mapping) is re-entered at §3.2 step 4 (pre-check by key), converging or landing in `pending_reconciliation`.

**Acceptance test for the guarantee:** inject a fault after every persistence boundary in §3.2's sequence; on restart the claim converges to exactly one financial outcome (§9 P0 tests).

### 3.4 Durable dispatch — transactional outbox (review P1)

A claim must not rely on Temporal being up at commit time. `POST /api/claims` commits the claim row **and an `workflow_outbox` entry** (`claim_id`, workflow id, state `pending`) in the same SQLite transaction. A small **dispatcher drainer** (runs in the worker process at startup and on a 5s cadence; also startable on-demand at `POST /api/demo/dispatch/run`) drains `pending` entries → `start_workflow(REJECT_DUPLICATE)` → mark `started`. Consequences:

- Temporal down at submission: claim is durably committed, `GET /api/claims/{id}` shows stage `awaiting_dispatch` with an "orchestrator offline" banner; the workflow starts automatically when Temporal returns. Submissions never lose a committed claim.
- SQLite committed, workflow-start failed or lost: recovered by the drainer (workflow id reuse policy makes re-starting a no-op-duplicate).
- Workflow progress that outpaces local state: activities are the only writer of local state and run after the corresponding workflow-history points — a lagging local read shows the DB truth, never worse than reality; reconciliation closes residual drift.

**Human review = durable wait via Workflow Update.** The workflow calls `wait_condition(review_decision is not None, timeout=review_timeout)`. Reviewer Approve/Reject/Override posts reach the live workflow as a Workflow Update `review-decision` with synchronous ack (`execute_update`). Update-handler rules (review P1):

- The handler rejects any second or contradictory decision (`decision already recorded`) — one valid terminal review decision per claim; the first decision in Temporal history order wins even when two reviewers submit simultaneously.
- A rejected concurrent attempt is audited (`review_conflict_rejected` with actor + attempt ts), never silently dropped.
- Timeout boundary: Temporal serializes the update and the timer expiry in workflow history — a decision that lands before the timeout transition completes is honored; after the timeout transition the claim sits in exception queue (`review_timeout`) but **the workflow keeps waiting**, so a late reviewer decision still resolves it and the queue entry auto-closes. Late decisions are audited as `late_review_decision`.

### 3.5 Reconciliation

**Temporal Schedule** (`reconciliation-every-2m` in dev): ingest settlement batch → match `provider_ref`/`idempotency_key` against instructions + ledger → deltas/unmatched into `exception_queue` + audit. Also re-queries `pending_reconciliation` instructions and closes `review_timeout` queue entries resolved late. "Matched" means provider-settlement record ↔ instruction ↔ ledger agree on key, amount and destination — it does **not** create financial moves; it asserts and reconciles. Schedule preferred over in-claim polling because it spans all claims, each run is independently visible in the Temporal Web UI, and pause/resume is free (demo control). On-demand: `POST /api/demo/reconciliation/run`.

**Windows/ops notes:** three consoles — (1) `temporal server start-dev --ui --port 7233 --db-filename ./data/temporal-dev.db`, (2) `uv run refund-worker`, (3) `uv run uvicorn novacart_refunds.api.main:app --reload --port 8000`. Never co-run the worker inside uvicorn `--reload` (double registration). First dev-server run downloads a few hundred MB.

## 4. Data model (SQLite; money = signed BigInteger paise)

| Table | Purpose / notable columns |
|---|---|
| `customers` | id, name, email, phone (masked in UI), kyc_status |
| `orders` | channel (quick_commerce/marketplace), status (incl. `failed_payment`), money breakdown (subtotal/discount/tax/delivery_fee/total), payment_txn_id |
| `order_items` | sku, qty_ordered/qty_delivered, unit/line money values |
| `payment_transactions` | method (upi/card/wallet), **`token_ref` only — no PAN column exists by design** (regulatory anchor in the schema), status (`captured|failed|settled|refunded_partial|refunded_full`), settlement_expected_days |
| `refund_claims` | claim_no (`NC-2026-0001`), **`request_id` (client idempotency key; unique where not null)**, scenario, item_ids JSON, evidence JSON, status/workflow stage, eligible_amount, risk_band, decided_by, decision_reason, `rule_versions` JSON, `tat_deadline` |
| `refund_reservations` | claim_id FK, amount, state (reserved/consumed/released), **scope_breakdown JSON (§3.1)**; partial unique index on claim_id WHERE state='reserved' (SQLite supports) — one reservation per claim |
| `refund_instructions` | claim_id FK, amount, destination_token_ref, **unique idempotency_key**, indexed provider_ref, state (created/submitted/accepted/rejected/timeout_ambiguous/**pending_reconciliation**/duplicate_detected/failed/credited), provider_response JSON, attempt_count, max_attempts |
| `ledger_entries` | signed amount; entry_type (reservation/refund_debit/penalty/adjustment/note); **unique (instruction_id, entry_type) where instruction_id not null — replay-proof postings (§3.3)** |
| `workflow_outbox` | claim_id, workflow_id, state (pending/started), created_at, started_at (§3.4) |
| `reviewers` | id, name, **role (reviewer/supervisor)**, demo token hash (§7 demo auth adapter) |
| `settlement_records` | provider_name, settlement_batch_id, provider_ref, instruction_id (nullable — unmatched is the demo case), amount, settled_at |
| `exception_queue` | kind (settlement_mismatch/unmatched_settlement/timeout_ambiguous/duplicate_attempt/**review_conflict**/review_timeout/provider_error/manual), ref, payload, state (open/auto_closed/resolved), resolution, **first_seen_at (aging)** |
| `risk_assessments` | assessor, model_version, score 0–100, band (low/medium/high), signals JSON (input-signal provenance), recommended_action, decision_threshold applied, reviewer override ref (nullable), **mandatory advisory note** ("Advisory only — not evidence of fraud") |
| `policy_rule_versions` | rule_set (fp/mi), semver version (unique per set), owner, **approval metadata (approved_by/approved_at, nullable ⇒ shown as mock placeholder)**, **effective_from**, state (draft/published/**withdrawn**), JSON definition (TAT days, caps, per-component allocation basis), sha256 |
| `audit_events` | ts, claim (nullable), stage 1–9, actor (system/customer/reviewer:X/demo_ai), event_type, payload JSON, rule_version, **prev_hash/self_hash — cheap tamper-evident chain (§7 verification, §12.3 assumptions)** |
| `customer_notifications` | claim, safe_status (BR-08 wording preserved verbatim), body, sent_at |

Pragmas: `foreign_keys=ON`, `journal_mode=WAL`, `busy_timeout=5000`. Sessions run short `session.begin()` transactions; the reservation guard runs `BEGIN IMMEDIATE` (§3.1). Hot indexes: claims(status), claims(order_id), audit_events(claim_id, ts), exception_queue(state), ledger(instruction_id).

**Policy lifecycle (review P1):** rule versions are immutable once published (sha256-checked at load); only one `published` version per rule_set may be *effective* at a decision timestamp (`effective_from` ordering); a `withdrawn` or not-yet-effective version ⇒ fail-closed (exception queue, never a silent default); every decision, audit event and notification carries the exact version used, so results are reproducible and placeholder status is never presented as approved policy.

## 5. Mock adapters (Protocols; concretions wired only in `adapters/registry.py`)

```python
class OrderSystemAdapter(Protocol):
    def get_order(self, order_id: str) -> OrderDetails: ...          # raises NotFoundError
    def get_fulfilment(self, order_id: str) -> FulfilmentInfo: ...   # delivered qtys, sensor flags
    def get_customer(self, customer_id: str) -> CustomerDetails: ...

class PaymentProviderAdapter(Protocol):
    def pre_check(self, idempotency_key: str) -> RefundResult | ProviderStatusUnknown: ...
    def execute_refund(self, request: RefundRequest) -> RefundResult: ...

class ProviderStatusUnknown(StrEnum):
    delayed        # provider eventually consistent — treat as unknown (§3.2), not as rejected
    unavailable    # provider unreachable — keep pending_reconciliation, retry same key under cap

class FailureMode(StrEnum):
    instant_accept; timeout_then_accept; reject; duplicate_callback; always_timeout; unavailable

class SettlementReportAdapter(Protocol):
    def fetch_batch(self, after_ts: datetime) -> list[SettlementRecord]: ...

class CustomerNotifier(Protocol):
    def send_claim_update(self, customer, claim, safe_status: str, fields: dict) -> None: ...

class RiskAssessor(Protocol):
    def assess(self, claim_ctx: ClaimContext) -> RiskAdvice: ...      # advisory only
```

The fake provider keeps an in-memory `dict[idempotency_key, outcome]` so replays yield `duplicate_detected`; its `FailureMode` is runtime-switchable via `POST /api/demo/payment-mode` — this is what makes timeout-then-accept demoable live. **Scope statement (review: the mock's limits are honest limits):** the in-memory key store resets on process restart, so it proves replay idempotency within a running provider instance, not across provider restarts. For the crash-injection restart tests (§9) a `persist=True` adapter backing the dict to `data/mock_provider.sqlite3` is used; the demo script itself runs against the standard in-memory mode. Activities receive adapters through the registry; tests inject stubs; no activity imports a mock directly.

## 6. Deterministic rules engine

Pure functions, zero AI input, **versioned in `policy_rule_versions`**, loaded **fail-closed** (unknown version ⇒ exception queue, never a silent default).

```python
def evaluate_policy(claim: ClaimContext, rules: RuleSet) -> PolicyOutcome
# eligible, breakdown: list[AmountLine], eligible_amount_paise, rule_version, notes, tat_deadline
```

- **`fp_v1` (failed payment):** eligible iff order.status == `failed_payment` AND txn.status == `captured` AND `token_ref` matches the original payment method (the regulatory anchor lives in *code*) AND the payment rail's TAT rule applies to the transaction's method. Amount = full captured amount. `tat_deadline = captured_at + (T+5 card/upi, T+1 wallet)` from rule JSON. ₹100/day penalty computed as a flaggable placeholder line. **Per the 2026 review: these are inputs pending Finance/Legal validation, not settled legal conclusions — the UI and audit label TAT and penalty as mock assumptions; eligibility is never presented as a legally guaranteed outcome, and the authoritative order/payment states + approved policy — not the bare `captured` flag — determine it.**
- **`mi_v1` (missing item):** missing qty from fulfilment adapter; per-line base = `unit_price_paise × qty_missing`.
- **`decide(policy_outcome, risk_advice, settings)`:** auto-approve only if eligible AND band ≠ high AND amount ≤ `auto_approve_cap_paise`; else reviewer queue or reject. Records the exact rule version + risk band behind every decision — reproducible. **AI invariant: the risk input influences only the routing (auto/process/review); the amount is computed before and independently of risk, so altering a risk assessment can never move a paise.**

### 6.3 Allocation & partial-refund accounting (review P0-4)

Component **allocation basis** is explicit in rule JSON (pending Finance/Legal, loudly flagged):

| Component | Allocation basis |
|---|---|
| item base price | direct — `unit_price × qty_missing` |
| order-level discount | proportional to line share of subtotal, capped at line value |
| order-level tax (CGST/SGST) | recomputed from the post-discount allocated line value, per rule version |
| delivery fee | full refund when any grocery line is missing (placeholder; flagged) |
| wallet credits / coupons / promotions | per rule JSON (`wallet_credit_first` ordering); **excluded from refund-back if rule says so (never refunded twice)** |

**Invariant B2 (component conservation & caps):** Σ over all claims of each component allocated to scope (P, S) ≤ approved component value; every allocated amount is non-negative where applicable; **repeated partial refunds on the same item draw down component caps, not just the item total** — an overlapping item claim or re-allocated tax can't re-award an already-refunded component (this is the scope-level guard in §3.1, working over component breakdowns persisted in each reservation).

**Rounding:** all arithmetic in integer paise; the proportional remainder goes to the highest-value allocated line; for successive partial refunds of the same pool, the *remaining* pool re-allocates from current balances (no re-rounding drift — property-tested: total refunded over any sequence equals what one final refund would have returned for the same aggregate quantities).

Every decision stores the calculated component breakdown lines and rule version used (audit-reproducible); unapproved rules stay visibly marked as demo assumptions in UI, API payload and audit — **never rendered as customer-approved policy**.

## 7. API surface & security

JSON error envelope (`errors.py`) with **documented schemas** (review: no bare strings): NotFoundError→404 `{code, message, ref}`; validation→422; `WorkflowAlreadyStarted`→**200 `already_active`** with `{claim_id, claim_status, safe_status}`; `OverlappingRefundError`→409 `{code: "duplicate_or_exhausted", claim_id, breakdown_ref}`; TemporalUnavailable→503 with retry hint.

**Client request idempotency (review P1) — distinct from provider idempotency:** `POST /api/claims` accepts `Idempotency-Key` (stored as `refund_claims.request_id`, unique). A retry after a lost response returns the **original claim** (`200 {"status": "replayed", "claim_id": ...}`), never a second claim. Provider idempotency keys are derived (`refund-{claim_id}-instr-{n}`) and never exposed as a substitute for client keys.

**Demo auth adapter (review P1):** seeded `reviewers` table + a local demo login issuing a short-lived signed token (`POST /api/auth/demo/login`, reviewer id + role). Reviewer routes (`approve|reject|override|exception-resolve`) require a valid authenticated identity with a **role check** — `override` and exception resolution require `supervisor`. The header `X-Reviewer-Id` is no longer trusted as identity by itself; unsigned header → 401 with a "run demo login first" hint. **Explicitly labeled demo-grade auth (single-node, no IAM):** the register in §12.3 records this as a demo assumption, not a security story.

**Audit chain (review P1):**
- **Canonical serialization:** event payload JSON with sorted keys; fixed ISO-8601 ts; hash = sha256 over `prev_hash + canonical(json)` — the canonical form is defined once in `audit.py` and used by both writer and verifier.
- **Concurrency-safe append:** the hash chain write happens inside the same `BEGIN IMMEDIATE`/short write transaction as the business change it audits, so appends serialize (SQLite lock) and `prev_hash` is read last-in-transaction.
- **Verification:** `GET /api/demo/audit/verify` walks the chain (optionally scoped to a claim) — detects any modified, deleted or truncated event, including the head. The **trusted head** is checkpointed to `data/audit_head.json` on each schedule tick; verification compares the walk's final hash against the checkpoint. **Demo assumption (§12.3):** the checkpoint is a local file, not WORM storage — it demonstrates tamper-*evidence*, not external anchoring.

Endpoints:

- `POST /api/claims` — submit claim (with `Idempotency-Key`), outbox commit + workflow start
- `GET /api/claims/{id}` — status, BR-08-safe customer status
- `GET /api/orders/{id}` — items/prior-claim hints for the submission form
- `GET /api/reviewer/queue`; `POST /api/reviewer/claims/{id}/approve|reject|override` — authenticated, role-checked, via Workflow Update
- `GET /api/claims/{id}/audit`; `GET /api/demo/audit/verify`; `GET /api/exceptions` + `POST /api/exceptions/{id}/resolve` (supervisor); `GET /api/kpi`; `GET /api/demo/stuck-claims` (recovery view)
- Demo: `POST /api/demo/seed` (idempotent), `POST /api/demo/payment-mode`, `POST /api/demo/risk-seed` (force high band), `POST /api/demo/reconciliation/run`, `POST /api/demo/dispatch/run`, `GET /api/demo/state`
- `POST /api/auth/demo/login`; `GET /healthz`, `GET /healthz/temporal`

Resilience: Temporal client calls go through a `temporal_proxy` helper — on RPC failure, read-only routes serve SQLite state and the UI shows an "orchestrator offline" banner; submissions commit + outbox (no 503 loss), workflow start deferred.

## 8. UI (Jinja2 + htmx, one stylesheet, no build step)

| Page | Contents |
|---|---|
| `/` claim submission | scenario toggle, seeded-order picker, item checkboxes/qty, evidence, submit → claim page |
| `/claims/{id}` | live timeline: 2s htmx poll showing stage, rule version, amount breakdown, advisory banner, instruction state (incl. `pending_reconciliation` plainly worded), notifications; polling self-stops on terminal status |
| `/reviewer` | queue with expandable evidence + rule breakdown + risk advisory; Approve/Reject/Override `hx-post` → row swap + "decision recorded" ack; conflict attempt shows "another reviewer decided"; SLA timer |
| `/claims/{id}/audit` | hash-chained events (actor, stage, rule version) + chain-verify banner + deep link into Temporal Web UI (:8233) |
| `/exceptions` | open exceptions incl. mismatch/timeout/duplicate/review-timeout, **aging (first_seen)**, resolve action (supervisor) |
| `/kpi` | STP %, turnaround, duplicate attempts prevented, reconciliation match % |
| recovery banner | `/claims` list surfaces `awaiting_dispatch` / stuck `pending_reconciliation` claims with a "run recovery" control |

## 9. Testing strategy (`uv run pytest` — green without a Temporal server)

**P0 — financial correctness across failure boundaries:**
1. Two distinct claims race for the same refundable balance (async sessions, real `BEGIN IMMEDIATE`) → accepted reservations never exceed the balance; loser gets 409 + BR-04 KPI +1.
2. Retry arrives after the provider accepts but before the response is persisted → replay converges; **exactly one** `refund_debit` (ledger unique index).
3. Process crash after provider acceptance, before ledger posting (fault injection at each persistence boundary) → worker restart converges to one outcome.
4. Duplicate + late + out-of-order provider callbacks → single financial effect; extras audited.
5. Two partial refunds overlap the same item & component caps → component conservation holds (B2).
6. Unknown policy version encountered during workflow recovery → fail-closed, exception queue, workflow blocked at stage 3 (never defaults).

**P1 — workflow, governance, reconciliation:**
7. Claim commits while Temporal is down → outbox drains and workflow starts when it returns; no lost claim.
8. Two reviewers submit conflicting decisions simultaneously → single terminal decision in history order; second audited as conflict.
9. Reviewer decision at the timeout boundary → exactly one resolution; late decision path audited.
10. Audit chain verified after deliberate modification + truncation → tampering detected against checkpoint.
11. Unauthorized/unauthenticated reviewer attempt on approve/override → 401/403, audited.
12. Reconciliation receives duplicate/late/out-of-order settlement records → no double-matching, exception aging correct.
13. Customer notifications stay accurate through ambiguous payment states (BR-08 golden-wording tests).

**Standing property/invariant assertions (all P0/P1 suites):**
- Money (B1): total refunded + outstanding reserved ≤ approved refundable balance, per scope.
- Execution: one instruction → ≤ one effective provider refund under the provider's documented idempotency contract.
- Ledger: one posting uniquely attributable to one instruction; replay adds nothing.
- Audit: every material transition has a verifiable event (actor, ts, rule version, financial refs).
- Recovery: app + worker restart neither loses nor silently duplicates an in-flight refund.
- AI: any risk-advice mutation changes routing only; the policy amount is bit-identical.

**Baseline suites (retained from v1):** allocation conservation + remainder policy (property-tested), fp TAT boundaries, decision gate, fail-closed registry; deterministic `FailureMode` matrix for `execute_refund`; API smoke with a monkeypatched Temporal client; optional live integration behind `--integration`.

## 10. Demo script (seeded world)

Seed (idempotent): 3 customers + reviewer identities; the 5-item quick-commerce grocery order with "Basmati Rice 5kg" missing; a UPI failed-payment order (₹1,999 captured, order not confirmed); a wallet failed-payment order (T+1 contrast); a **prior claim by the same customer** (already refunded, provider_ref known); a settlement batch with one deliberate mismatch.

| Beat | Shows |
|---|---|
| A — failed payment STP | stages 2–6 auto-approve (fp_v1, T+5 deadline displayed, "advisory: low risk 0.12"), execute → close; audit hash chain + Temporal UI |
| A2 — timeout-then-accept | mock mode flipped live; `timeout_ambiguous` → `pre_check` → `duplicate_detected` → credited, exactly one ledger debit |
| A3 — forced high risk | `risk-seed` → claim routes to reviewer despite policy eligibility; AI advisory, deterministic decision; demo-login + role check shown |
| B1 — duplicate resubmission | prior claim resubmitted → REJECT_DUPLICATE + "already active" banner, KPI +1, zero paise moved |
| B2 — real missing-item claim | item-level breakdown with proportional discount/tax allocation + flagged delivery-fee placeholder; reviewer approves; settlement; mismatch lands in `/exceptions`; resolve on screen; close on `/kpi` |
| B3 — overlapping item claim *(new)* | second claim attempts the same missing item → 409 `duplicate_or_exhausted` at reservation, component caps unchanged |
| A4 — recovery *(new)* | kill worker mid-execution in `timeout_then_accept`; restart → converges to `credited`; audit shows the full convergent path |

## 11. Build order (milestones → implementation plan)

- **M0** Scaffold: verify `temporalio` cp314 wheel on Windows (else Python 3.13), import smoke, reproducible tests.
- **M1** Contracts & data: schema (incl. request_id, scope_breakdown, ledger unique index, outbox, reviewers), canonical audit serialization, `BEGIN IMMEDIATE` reservation guard skeleton.
- **M2** Domain (pure; no Temporal): versioned policies, allocation with component caps, decision gate, `balances.py` computation, property-based tests.
- **M3** Temporal + recovery: outbox dispatcher, review-decision race handling, §3.2/§3.3 execution sequence with `pending_reconciliation`, restart sweep.
- **M4** API & security: error schemas, client idempotency, demo auth adapter + roles, audit verify endpoint.
- **M5** UI & reconciliation: authoritative statuses + safe wording, exception aging, recovery views.
- **M6/M7** Prove the guarantees: concurrency, crash-injection, replay, audit-tamper, and end-to-end demo runs before declaring the BRD controls demonstrated.

## 12. Risks & demo-assumption register

1. **temporalio ↔ Python 3.14** — supported ≥1.34 (verified); live M0 check; fallback Python 3.13.
2. **SQLAlchemy/pydantic-settings on 3.14** — import smoke in M0; pin SQLAlchemy ≥2.0.44 if needed.
3. **Demo-assumption register (review-driven honesty):** the following are deliberately scoped for the demo and must never be presented as production guarantees — (a) `X-Reviewer-Id`-grade auth replaced by a demo login adapter: still demo-grade, not IAM; (b) audit trusted head is a local file checkpoint — tamper-evidence, not WORM anchoring; (c) `persist=True` mock-provider backing exists only for crash tests; (d) TAT deadlines/₹100/day penalty are mock inputs pending Finance/Legal validation, never presented as settled legal conclusions; (e) SQLite single-node — serialized `BEGIN IMMEDIATE` is correct here, a production deployment needs a fresh capacity/availability/backup/concurrency assessment.
4. **Workflow sandbox determinism** — no `random`/`uuid4`/wall-clock in workflow code; use `workflow.uuid7()`/`workflow.now()` or derive ids/timestamps in activities; `shared.py` stays free of SQLAlchemy imports.
5. **Windows** — dev-server download on first run; `--db-filename` so history survives restarts; never run the worker inside uvicorn `--reload`; PowerShell env quoting in scripts.
6. **SQLite two-process concurrency** — WAL + `busy_timeout` + short transactions; the reservation guard in one `BEGIN IMMEDIATE` transaction; verify write-lock contention under the concurrency tests (§9 P0-1) rather than assuming it passes.
7. **Update semantics** — confirm `execute_update` synchronous ack + update-vs-timer serialization when building M3/M4 (Temporal guarantees a single history order — assert it with test 9).
8. **Money rounding** — explicit remainder policy, property-tested across repeated partial refunds (§6.3).
9. **Ops/observability** — structured logs keyed by claim_no correlation id; retry counters on instructions + recon runs; workflow failure alerts (worker log + exception queue); exception aging on `/exceptions`; recovery runbook: `/api/demo/stuck-claims` → dispatch-run → recon-run; escalated stuck claims (e.g. `pending_reconciliation` past `max_attempts`) surfaced for manual supervisor action.
10. **Scope guard** — marketplace settlement/TCS stays out of v1; `settlement_records.provider_name` kept generic for later extension.