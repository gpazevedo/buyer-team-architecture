# AD-016 — REQUIRES_ATTENTION with Twenty Typed Triggers

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-16 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-17, AD-12

## Context

Many distinct conditions warrant pulling a negotiation out of automated flow — zero bids, risk breach, approval timeout, config unavailability, delivery failures, token-ceiling abuse, total timeout, and more. Operations needs to filter and triage these on a dashboard, which requires a structured taxonomy rather than a single opaque "failed" state.

## Decision

Use one `REQUIRES_ATTENTION` status reachable from *any* status, carrying a machine-readable `entry_trigger` code plus a human-readable `entry_reason`. Maintain a numbered trigger table (currently 20 — see the 2026-08-05 count reconciliation in Results) where each trigger has a condition, escalation path, and SLA. Sub-codes (e.g. `superseded_marking_failure`) are emitted as the literal `entry_trigger` value so dashboards can filter at sub-code granularity, while the column-3 code remains the canonical taxonomy mapping. Ops always resolves to either ACTIVE or CANCELLED.

## Alternatives Considered

- **A distinct status per failure type.** Rejected: explodes the state machine and complicates every transition guard; the trigger code carries the discriminator instead.
- **A single untyped "error" flag.** Rejected: ops cannot triage or filter; no per-condition SLA.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Ops dashboards filter at sub-code granularity with per-condition SLAs | The trigger table is a living maintenance surface — it has grown 9 → 13 → 14 → 16 → 17 → 18 → 19 → 20 across revisions, and two of those growth steps collided on the same number |
| Core state machine stays small (one status, not one per failure type) | Each new failure mode discovered in audit must be slotted in with a code, SLA, and escalation path |

Keeping the state-machine diagram and cross-PRD trigger-count references in sync is recurring coherence work.

## Results

Ops dashboards filter at sub-code granularity; every escalation has an SLA and a defined resolution. The single status keeps the core state machine small while the trigger code carries the detail. `REQUIRES_ATTENTION` deliberately does **not** decrement the tenant concurrency counter, because the negotiation may resume to ACTIVE without re-admission. A 7-day stall in REQUIRES_ATTENTION auto-cancels (REQ-G203a) so nothing stalls forever.

**Correction 2026-08-04 (impl PR #256): the sentence "A 7-day stall in REQUIRES_ATTENTION auto-cancels (REQ-G203a) so nothing stalls forever" was a specification, not a behaviour — and the opposite was true.** REQ-G203 (96h in PENDING_APPROVAL → REQUIRES_ATTENTION, trigger #3 `approval_timeout_96h`) and REQ-G203a (7 days under that trigger → CANCELLED) were both written into PRD-001 REQ-505 and PRD-002 §4.4 from the start, but neither had an implementation: nothing swept for either condition, so a negotiation could sit in PENDING_APPROVAL indefinitely and trigger #3 was never emitted by anything.

Worse, the Step Functions gate actively defeated the ceiling it was supposed to enforce. `ApprovalGate` carried `States.Timeout` in its retry list, so when the 96h `TimeoutSeconds` fired, Step Functions re-invoked Node 6 — whose `_pause_for_human` resets `approval_started_at` and installs a fresh task token, re-arming the clock from zero. Each retry bought another 96 hours, and the 96h ceiling this trigger encodes never actually bounded anything.

PR #256 makes both durable via a scheduled sweep rather than the state machine (AD-134): `States.Timeout` is removed from the gate's retry list so the execution ends cleanly at an `ApprovalTimedOut` Succeed state, and a 30-minute `requires_attention_evaluator` Lambda performs the status writes. The trigger *taxonomy* is unchanged by this — `approval_timeout_96h` was always trigger #3 and the count is unaffected; what changed is that trigger #3 now fires. Note the resulting SLA shape: escalation is bounded by 96h + one sweep interval, so the trigger's own SLA is met on the next 30-minute boundary rather than at the instant of expiry.

**Count reconciliation 2026-08-05 — the taxonomy is 20 triggers, and two of them were both numbered #19.** This ADR's title said eighteen, its Decision said "currently 18", and the note below claimed `compensation_incomplete` as the nineteenth. Counted against the canonical table in PRD-002 §4.4, all three were wrong:

- The PRD table holds **19 numbered rows**, and **#19 is `kraljic_low_confidence`** (PR #216) — a trigger this ADR had never mentioned. It is live in `orchestrator/node_kraljic_classify.py`.
- **`compensation_incomplete` is the 20th**, not the 19th. It was assigned #19 here without checking that PR #216 had already taken that number.
- `compensation_incomplete` is **absent from PRD-002 §4.4 entirely** (zero occurrences in PRD-002 and PRD-006), so the canonical table does not yet carry the trigger this ADR has documented since 2026-07-16. Both codes are real in impl; it is the spec table that is behind.

Title and Decision now say twenty. Renumbering the PRD table and adding the missing row is a PRD-side fix, tracked there rather than resolved here — this ADR owns the decision, not the table.

This is the concrete form of the maintenance cost the Trade-offs table warns about, and it is worth naming precisely: the failure was not that a trigger went undocumented, but that **two independent authors each appended "the next one" to a table they read at different times**. A count in prose cannot detect that; only the numbered table can, and only if it is the single place numbers are assigned.

**20th trigger — `compensation_incomplete` (PR #220, merged 2026-07-16).** Recovery's total-timeout path (`orchestrator/resilience/recovery.py::run_recovery_flow`) previously cancelled a timed-out negotiation unconditionally once its saga compensation ran, regardless of whether every compensating action actually succeeded — a partial compensation failure (e.g. a still-ISSUED PO a supplier-side compensating call couldn't undo) was silently orphaned by a `CANCELLED` write, terminal and excluded from `RECOVERABLE_STATUSES`, so it could never be retried. Recovery now branches on `compensate_negotiation`'s result: full success still cancels as before, but any un-compensated entries route to `REQUIRES_ATTENTION` with `entry_trigger="compensation_incomplete"` instead, keeping the negotiation on the recovery sweep so a future sweep retries the outstanding saga-log entries. Field convention mirrors `node_strategy_execute.py`'s own `_flag_requires_attention`.

**Wiring closure 2026-08-14 (impl PR #293).** A 2026-08-13 audit of PRD-002 §4.4 found there was no `evaluate_triggers()` dispatcher anywhere in the repo — escalation is, and remains, decentralized per-node — and, worse, that several triggers already marked "confirmed live" in the audit table had been classified that way by grepping for the string `entry_trigger`, which does not prove a `NEGOTIATIONS_TABLE.update_item()` call actually fires. Trigger **#9 `communication_delivery_failure`** was the sharpest case: `node_award_comms.py` returned a `REQUIRES_ATTENTION`-shaped dict on a comms failure, but nothing ever persisted it and the Step Functions `AwardComms` state went straight to a bare `Succeed` — a delivery failure ended the execution cleanly with zero trace in the negotiation's row. PR #293 fixes that and wires eight more previously-inert triggers to a real `entry_trigger` write: #6 `retry_exhaustion`, #8 `configuration_unavailable`, #10 `po_assembly_failure`, #11 `convergence_detection_unavailable`, #13 `auction_round_delivery_failure`, #14 `proposal_delivery_failure`, and the token-ceiling pair #15/#17. #16 `negotiation_total_timeout`'s dead `check_negotiation_timeout` wrapper (never called anywhere) was deleted as superseded by #20 `compensation_incomplete`. Two triggers were resolved as deliberately **not** a per-negotiation write: #4 `fallback_quality_below_threshold` is a rolling-window CloudWatch alarm on the already-emitted `fallback_rejection_count` metric (the spec's own wording is "rolling-window alarm", not a row-level escalation), and #7 `recovery_lock_timeout` is a documented no-op — taking over an expired lock during a recovery sweep *is* the recovery, so there is nothing to escalate. With this PR, all 20 triggers in the taxonomy have a resolved, verified fate — either a real DynamoDB write or an explicit, documented reason why one doesn't apply.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
