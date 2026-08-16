# AD-134 — A Scheduled Sweep, Not the State Machine, Owns the Approval Timeout

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-134 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-2, AD-16, AD-17, AD-15

## Context

Node 6 pauses a negotiation for human approval using a Step Functions `waitForTaskToken` task with `TimeoutSeconds` of 96 hours. Two requirements sit on that pause, both specified since PRD-001 REQ-505 and PRD-002 §4.4: **REQ-G203**, a hard 96h ceiling after which the negotiation goes to REQUIRES_ATTENTION under trigger `approval_timeout_96h` with no auto-approval; and **REQ-G203a**, an auto-cancel after a further 7 days of ops inaction, so nothing stalls forever.

Neither was implemented, and the state machine as configured made the first one unenforceable. `ApprovalGate` carried `States.Timeout` in its shared retry list, so when the 96h timeout fired, Step Functions retried the task — which re-invoked Node 6, whose `_pause_for_human` writes a fresh `approval_started_at` and installs a new task token. Every timeout bought another 96 hours from a clock that had just been reset. The ceiling this trigger encodes never bounded anything, and because each retry looked like a normal pause, nothing in the execution history flagged it.

Fixing the retry alone is not sufficient. Step Functions can end an execution on timeout, but it has **no hook that writes the negotiation's status row** — and that row, not the execution, is what the app reads (AD-17: the DynamoDB status write is authoritative). An execution that simply ends on timeout leaves the negotiation sitting in PENDING_APPROVAL forever with nothing left running to move it. REQ-G203a compounds this: its 7-day window begins *after* the negotiation has entered REQUIRES_ATTENTION, an ops-owned state the original execution has already exited, so no single execution can own both deadlines even in principle.

## Decision

Split the responsibility. The state machine owns *ending the pause*; a scheduled sweep owns *the status transitions*.

`States.Timeout` is removed from the ApprovalGate's retry list (derived from the shared `task_retry` local, so ordinary Lambda-exception retry tuning stays in one place) and caught into a new `ApprovalTimedOut` `Succeed` state. A timed-out approval is a valid outcome of the workflow, not a failure, and the execution ends cleanly instead of re-arming the clock.

A new `requires_attention_evaluator` Lambda — `{env}-buyer-team-approval-sweep`, EventBridge `rate(30 minutes)` via `var.approval_sweep_schedule`, its own IAM role — performs both transitions on its next tick:

- **G203** — query the `status_index` GSI for `PENDING_APPROVAL`, and for any row aged ≥ 96h (from `approval_started_at`, falling back to `created_at`) do a **conditional** `UpdateItem` to REQUIRES_ATTENTION with `entry_trigger="approval_timeout_96h"` and `ra_entered_at`, guarded by `ConditionExpression="#s = :pending"`. Then mirror the status onto the app-visible requisition row (REQ-G302), emit `requires_attention`, and best-effort `send_task_success` on the stored task token.
- **G203a** — query the same GSI for `REQUIRES_ATTENTION`, and for rows whose `entry_trigger` is `approval_timeout_96h` and whose `ra_entered_at` is older than 7 days, conditionally transition to CANCELLED with `cancellation_reason="approval_stale_7day_limit_exceeded"`, record the terminal for REQ-O223 re-baselining, emit `approval_stale_7day_auto_cancel`, and close the PR cycle-time KPI as `cancelled`.

The conditional write is what makes the sweep safe to run on a timer: a human approval landing between the sweep's read and its write wins, and the sweep's update becomes a no-op rather than clobbering it. The same property makes the sweep idempotent across overlapping ticks. Per-item failures are logged and the sweep continues — one bad negotiation must not strand the rest.

The task-token resolve is deliberately best-effort. The sweep always runs *after* the SFN's own 96h timeout has already ended the execution, so the token is normally dead; a `ClientError` there is logged at INFO, not raised.

## Alternatives Considered

- **Keep the `States.Timeout` retry and teach Node 6 to detect re-entry.** Rejected: Node 6 would have to distinguish a fresh pause from a timeout re-entry, and even then Step Functions still has no way to write the authoritative status row. It preserves the clock-reset bug one refactor away from returning.
- **A per-negotiation EventBridge Scheduler one-shot timer at +96h.** Rejected: one schedule per pending approval, each needing cleanup on the happy path — a resource-count leak proportional to the pending population, where a single sweep is O(1) infrastructure regardless of it.
- **DynamoDB TTL on the negotiation row.** Rejected: TTL deletes an item, it cannot transition one, and its delivery is best-effort within 48 hours — an order of magnitude outside the SLA this trigger carries.
- **A `Wait` + `Choice` polling loop inside the execution.** Rejected: it could express the 96h ceiling, but not REQ-G203a, whose 7-day window runs after the execution has ended. Two deadlines spanning an execution boundary cannot both live inside one execution.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The 96h ceiling is durable and actually bounds — a negotiation can no longer stall in PENDING_APPROVAL indefinitely | Escalation is bounded by 96h **+ one sweep interval**, not exact; trigger #3's SLA is met on the next 30-minute boundary rather than at the instant of expiry |
| One sweep handles any pending population — no per-negotiation infrastructure to provision or leak | Two full GSI queries per tick over the PENDING_APPROVAL and REQUIRES_ATTENTION populations; comfortable at dev volume, a scan-shaped cost that will need a sparse index or an age-keyed GSI at scale |
| G203a can reach negotiations whose Step Functions execution has already ended — impossible from inside the workflow | The status transition now happens outside the state machine, so the execution history no longer tells the whole story of a negotiation's lifecycle; two places must be read to reconstruct it |
| Conditional writes make the sweep race-safe against concurrent human approval, and idempotent across ticks | One more always-on scheduled Lambda and IAM role, alongside `runtime_warmer`, `dlq_redrive`, `recovery`, `heartbeat`, `outbox_poller`, and the daily rollup/cost pollers |

This is the same shape as AD-2's split — Step Functions owns what fits inside an execution, DynamoDB plus a sweep owns what outlives one — applied to a deadline rather than to session state.

## Results

Shipped in impl PR #256 (merged 2026-08-04): `orchestrator/requires_attention_evaluator.py`, the `approval_sweep` Lambda, IAM role and EventBridge rule in `infra/modules/step-functions/main.tf`, and the `approval_retry` / `ApprovalTimedOut` change to the ApprovalGate state.

**Interaction with the pre-existing recovery sweep.** Two independent sweeps now write REQUIRES_ATTENTION rows: this one and `resilience/recovery.py`'s total-timeout path (AD-16's 19th trigger). They do not collide, and the reason is load-bearing rather than incidental — G203a filters on `entry_trigger == "approval_timeout_96h"`, so a negotiation escalated by recovery under `compensation_incomplete` is invisible to the auto-cancel and stays on the recovery sweep where it belongs. The typed-trigger taxonomy (AD-16) is what makes two sweeps over one status safe. A moto-backed characterization suite exercises the interaction directly.

**A boto3 gotcha this decision paid for twice.** Both this evaluator's `_query_status` and `run_recovery_flow` originally took their low-level client from `resource.meta.client`. That client still carries the resource layer's attribute-value injector, so a manually-formatted `ExpressionAttributeValues` (`{":status": {"S": ...}}`) gets serialized a second time and matches nothing. Both now build `boto3.client("dynamodb")` directly. The failure was invisible to the test suite because the in-memory `_FakeTable` never validated `ConditionExpression` placeholders — a `:pending`/`:ra` typo shipped with everything green. The fix is the moto-backed suite that exercises the real conditional-write contract; the general lesson is that a fake which accepts any expression cannot test a mechanism whose entire correctness *is* the expression.

**Related fix on the same pause path (impl PR #257).** `_pause_for_human` wrote `approval_started_at` and the requisition-status mirror but never set the Award row's `approval_status = PENDING_HUMAN`, leaving one of the four award-status transitions unwired while the other three were correct. Same class of gap as the one this ADR closes — a status surface a reader depends on, not written by the code that owns the transition — which is why the pause needed two separate PRs to become fully observable.

**Not yet live-verified.** The sweep is deployed and unit/moto-tested, but no negotiation has been observed crossing 96h or 7 days on dev — both windows exceed any test run to date, and forcing one requires back-dating `approval_started_at` on a live row.

**Update 2026-08-16 (impl PR #319): the sweep failed every 30-minute invocation for its first 12 days — silently.** From 2026-08-04 (the day PR #256 shipped) to 08-16, every tick died with `AccessDeniedException` on its `status_index` GSI query — the IAM role lacked the Query grant on the index ARN — and nothing surfaced it, because no Lambda-error alarm existed. Fixed: the grant, plus `approval_sweep_errors` (`AWS/Lambda Errors >= 1`, wired to `evaluation_alerts`) so the failure mode is loud next time. A related mirror gap closed in the same PR: Node 5's RA escalations (`suspicious_bid_overflow`, `risk_threshold_breach`) previously mirrored only the negotiation row, and since this sweep's mirror covers only PENDING_APPROVAL rows, the app-visible requisition stayed IN_NEGOTIATION forever — `_flag_requires_attention` now writes the requisition row too.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
