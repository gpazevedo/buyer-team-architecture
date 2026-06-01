# AD-084 — Reconciler Lambda Corrects Counter Drift

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-84 · **Source PRD:** PRD-016 · **Status:** Accepted · **Related:** AD-81, AD-16

## Context

In-flight counters are maintained transactionally on the happy path (AD-81), but crashes can bypass the decrement: a runtime that dies after completing work but before executing the terminal-status transaction leaves the counter permanently incremented. Over time, leaked counters reduce a tenant's effective available capacity. The concurrency counter has a ground truth (the set of non-terminal negotiations in DynamoDB) against which it can be compared and corrected.

## Decision

Run a `{env}-tenant-concurrency-reconciler` Lambda on a 5-minute schedule. It scans non-terminal negotiations in `{env}-negotiations`, computes observed in-flight per tenant, and corrects the stored counter via conditional `UpdateItem` when drift is detected. It alarms when drift exceeds 10% or 5 absolute. It reconciles only the concurrency counter — the cost-budget counter has no observable ground truth and fails safe by over-restricting, so it is excluded.

## Alternatives Considered

- **Rely solely on the transactional path; accept leaked counters.** Rejected: crash paths are realistic and leaked counters permanently reduce effective capacity without self-correction.
- **Reconcile continuously (stream-driven).** Rejected: DynamoDB Streams would add latency complexity; a 5-minute periodic scan is sufficient given the drift-to-impact timeline and is simpler to reason about.
- **Reconcile both the concurrency and cost-budget counters.** Rejected: the cost-budget counter has no ground truth (cost accrual is asynchronous and approximate), so correcting it could flip it in either direction; failing safe by over-restricting is preferable.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Counter drift from crash paths is self-corrected within ~5 minutes, preventing permanent capacity reduction | Eventual rather than instant consistency: drift persists for up to one reconcile interval before correction |
| Drift is observable (`tenant.counter_drift` metric) and alarmed, making capacity anomalies visible | Periodic scan over `{env}-negotiations` incurs read cost every 5 minutes |
| Reconciler uses optimistic concurrency, so it cannot clobber a concurrent in-flight admission | The reconciler must itself be idempotent; any bug in it becomes a counter-corruption vector |

## Results

Corrections are conditional `UpdateItem` writes (optimistic concurrency). Interacts cleanly with PRD-006 crash recovery (the slot is never released during a crash, so resumption needs no re-admission). For a permanently abandoned negotiation, the 168h total-timeout → `REQUIRES_ATTENTION` transition (AD-16) holds the slot until ops cancels it — at which point the standard terminal decrement fires. The 168h timeout-into-REQUIRES_ATTENTION and AD-16's 7-day in-state auto-cancel are distinct timers that compose.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
