# AD-082 — Spend Gate Is a Non-Atomic Pre-Check with Bounded Overshoot

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-82 · **Source PRD:** PRD-016 · **Status:** Accepted · **Related:** AD-81, AD-61, AD-83

## Context

A per-tenant absolute spend cap is needed: a tenant with abusive behavior from onboarding sets a high baseline and never trips the relative anomaly alarm, so a preventive hard cap is the only structural stop. However, accrued cost is known only after invocations complete and can only be compared against a config value by reading an aggregated counter — an operation that cannot be expressed as a DynamoDB transaction condition. The spend check therefore cannot join the atomic admission transaction of AD-81.

## Decision

Make the spend gate a non-atomic `Query` on the near-real-time accrued-cost counter executed *before* the capacity transaction (AD-81). Accept a bounded overshoot of `max_slots × per-negotiation ceiling` — approximately $100–120 at defaults (20 slots × ~$5–6/negotiation). Budget rejections set `Retry-After` to the window boundary (top of next hour or midnight UTC), not the capacity retry-ladder, because routing budget-blocked work through the 5-retry→DLQ ladder would exhaust the ladder within ~25 minutes — long before the budget clears. Read the estimated near-real-time counter (not the CUR table, which lags hours) to keep the gate timely.

## Alternatives Considered

- **Include the budget check in the admission transaction (AD-81).** Rejected: a budget-vs-config comparison requires reading an aggregated counter and comparing to a config value, which cannot be expressed as a DynamoDB `ConditionCheck` expression.
- **Use the CUR-joined cost table (AD-61) as the authoritative source.** Rejected: CUR lags hours, making the gate stale and potentially allowing substantial overspend before a rejection fires.
- **Route budget rejections through the standard capacity retry-ladder.** Rejected: the ladder exhausts (5 retries → DLQ) in ~25 minutes, far before a budget window boundary; work would be discarded rather than held.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Preventive absolute spend cap that complements the detective anomaly alarm and per-session output-token ceiling | Exactness and atomicity: a small TOCTOU race between the pre-check read and the admission transaction is possible |
| Near-real-time enforcement using the estimated counter keeps the gate timely | The overshoot envelope (~$100–120 at defaults) means the cap is soft within a closed-form bound, not a hard stop |
| Distinct `Retry-After` semantics (window boundary, not retry-ladder) avoids DLQ churn for budget-blocked tenants | |

The overshoot is bounded and closed-form because the PRD-005 per-session output-token ceiling caps per-negotiation cost; without that ceiling the overshoot would be unbounded.

## Results

Reads `{env}-tenant-cost-budget` (self-expiring TTL buckets keyed to the budget window; no reset cron needed). Enforces a logical-OR of hourly and daily breach. The cost-budget counter is not reconciled by AD-84 — it fails safe by only ever over-restricting. Kill-switch separation is owned by AD-83 (`tenant_budget_enforcement` defaults to shadow mode so the gate emits metrics but does not block until explicitly enabled).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
