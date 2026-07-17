# AD-016 — REQUIRES_ATTENTION with Eighteen Typed Triggers

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-16 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-17, AD-12

## Context

Many distinct conditions warrant pulling a negotiation out of automated flow — zero bids, risk breach, approval timeout, config unavailability, delivery failures, token-ceiling abuse, total timeout, and more. Operations needs to filter and triage these on a dashboard, which requires a structured taxonomy rather than a single opaque "failed" state.

## Decision

Use one `REQUIRES_ATTENTION` status reachable from *any* status, carrying a machine-readable `entry_trigger` code plus a human-readable `entry_reason`. Maintain a numbered trigger table (currently 18) where each trigger has a condition, escalation path, and SLA. Sub-codes (e.g. `superseded_marking_failure`) are emitted as the literal `entry_trigger` value so dashboards can filter at sub-code granularity, while the column-3 code remains the canonical taxonomy mapping. Ops always resolves to either ACTIVE or CANCELLED.

## Alternatives Considered

- **A distinct status per failure type.** Rejected: explodes the state machine and complicates every transition guard; the trigger code carries the discriminator instead.
- **A single untyped "error" flag.** Rejected: ops cannot triage or filter; no per-condition SLA.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Ops dashboards filter at sub-code granularity with per-condition SLAs | The trigger table is a living maintenance surface — it has grown 9 → 13 → 14 → 16 → 17 → 18 → 19 across revisions |
| Core state machine stays small (one status, not one per failure type) | Each new failure mode discovered in audit must be slotted in with a code, SLA, and escalation path |

Keeping the state-machine diagram and cross-PRD trigger-count references in sync is recurring coherence work.

## Results

Ops dashboards filter at sub-code granularity; every escalation has an SLA and a defined resolution. The single status keeps the core state machine small while the trigger code carries the detail. `REQUIRES_ATTENTION` deliberately does **not** decrement the tenant concurrency counter, because the negotiation may resume to ACTIVE without re-admission. A 7-day stall in REQUIRES_ATTENTION auto-cancels (REQ-G203a) so nothing stalls forever.

**19th trigger — `compensation_incomplete` (PR #220, merged 2026-07-16).** Recovery's total-timeout path (`orchestrator/resilience/recovery.py::run_recovery_flow`) previously cancelled a timed-out negotiation unconditionally once its saga compensation ran, regardless of whether every compensating action actually succeeded — a partial compensation failure (e.g. a still-ISSUED PO a supplier-side compensating call couldn't undo) was silently orphaned by a `CANCELLED` write, terminal and excluded from `RECOVERABLE_STATUSES`, so it could never be retried. Recovery now branches on `compensate_negotiation`'s result: full success still cancels as before, but any un-compensated entries route to `REQUIRES_ATTENTION` with `entry_trigger="compensation_incomplete"` instead, keeping the negotiation on the recovery sweep so a future sweep retries the outstanding saga-log entries. Field convention mirrors `node_strategy_execute.py`'s own `_flag_requires_attention`.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
