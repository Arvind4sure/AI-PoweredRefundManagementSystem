# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status: Discovery Phase — No Code Yet

This repository currently contains **only project documents** for an AI-Powered Refund Management System, a mock use case for a fictional client, **NovaCart Commerce** (an India-based omnichannel e-commerce platform: marketplace + quick-commerce dark stores, prepaid via UPI/cards/wallets). There are no build, lint, or test commands yet. When application scaffolding is added, add the actual commands here.

The project owner is an **FDE (Forward Deployed Engineer) team**; the BRD is explicitly a *mock customer-discovery draft (v0.1, not approved)* — numeric targets, policies, and integrations are assumptions pending client validation. Project plan spans Q4 2026 → Q3 2027.

## Source Documents

All under `Use Case 1 - Refund Management/`:

| Document | Path | What it governs |
|---|---|---|
| Company profile | `Company profile/NovaCart Commerce - Company Profile.pdf` | Business context, refund lifecycle, KPIs, definition of success |
| Government policies | `Company profile/Government Policies/NovaCart_Refund_Government_Policies.pdf` | Regulatory constraints (India): Consumer Protection Act 2019 / E-Commerce Rules 2020, RBI refund TAT (failed payments: card/UPI T+5, wallet T+1, ₹100/day auto-reversal penalty), RBI Payment Aggregator Directions 2025 (refund to original payment method, no card-number storage — tokens only), CGST §34 (credit notes) & §52 (marketplace TCS), DPDP Act 2023, FSSAI shelf-life (30% or 45 days) |
| Company policies | `Company profile/Company Policies/Company Policies.pdf` | Platform ToS, vendor/merchant agreements, dark-store SLAs, RTO cost ownership, last-mile OTP liability, escrow/nodal accounts |
| BRD | `Company profile/NovaCart_Refund_Management_BRD_Mock_Draft.docx` | Requirements BR-01…BR-11, business process, open questions (OQ-01…OQ-07), discovery question list |
| Project timeline | `UseCase Project Documents/Project timeline.xlsx` | Phased plan (conception → close) |

## The Refund Lifecycle (the system's backbone)

Any implementation should map to these 9 stages from the BRD §7:

1. Request intake (order ID, item, quantity, reason, resolution, evidence)
2. Identity & transaction verification (customer, order, payment, fulfilment)
3. Policy & eligibility validation → eligible amount
4. Duplicate & financial checks (prior claims, reservations, remaining refundable balance)
5. AI-assisted risk & evidence assessment (advisory only)
6. Decision & approval (auto-process only within approved policy/risk limits)
7. Refund execution (idempotent, through payment integrations)
8. Settlement & reconciliation (provider ↔ ledger ↔ settlement; mismatches → exception queue)
9. Customer notification & closure

## Non-Negotiable Design Constraints

These come straight from the BRD/profile and should shape every implementation decision:

- **AI is advisory; deterministic rules are authoritative.** The AI model must never independently determine eligibility, calculate authoritative refund amounts, or define financial allocation rules. Finance & Legal own the allocation rules (discounts, coupons, taxes, delivery fees, wallet credits, promotions, seller settlements).
- **Duplicate/concurrency protection is a "Must" (BR-04).** No refund may exceed the remaining eligible amount; concurrent requests must never cause duplicate financial execution; in-flight refunds are reserved.
- **Idempotent execution with safe handling of success/failure/timeout/retry (BR-06)** — e.g. a provider that accepted a request but timed out must not be retried blindly.
- **End-to-end auditability (BR-09):** every evidence item, rule version, decision, approval and financial event is traceable from original order to final financial outcome.
- **Customer-facing honesty (BR-08):** statuses never promise unverified settlement dates.
- **Risk routing:** high-risk/ambiguous/exception cases go to authorized human reviewers with sufficient evidence (BR-05); a risk score is not proof of fraud.
- **Regulatory anchors in code, not comments:** refund-to-original-payment-method, tokenized card references (never raw PANs), 48h grievance ack / 1-month resolution, no-refusal for defective goods, DPDP data-handling duties.

## Refund Scenarios to Support

Failed payments (customer debited, order not confirmed), cancellations, missing items (item-level partial refunds), incorrect deliveries, damaged goods, quality issues, returns — across both **marketplace** (seller settlements, TCS) and **quick-commerce/dark-store** (weight-sensor verification, missing-item liability) flows.