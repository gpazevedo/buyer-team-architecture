# AD-015 — Concurrent Recovery Lock Scoped to (tenant_id, negotiation_id), 600s TTL

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-15 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-14

## Context

After a Runtime restart, more than one instance may attempt to recover the same in-progress negotiation. Two instances replaying the same checkpoint concurrently would race on state writes and side effects.

## Decision

Guard recovery with a DynamoDB conditional-write lock keyed on `(tenant_id, negotiation_id)`. The TTL is **600s (10 minutes)** — sized to exceed worst-case single-node execution: A2A timeout (120s) × max_retries (up to 3) + backoff ceiling + checkpoint write budget ≈ 500s, then rounded up (REQ-G301b). If the lock holder crashes before releasing, TTL expiry lets another instance acquire it.

## Alternatives Considered

- **No lock.** Rejected: concurrent recovery corrupts state.
- **Short TTL (e.g. the original 5min).** Rejected and superseded in v1.0.4: too short to cover a node mid-execution, so a slow-but-healthy holder would have its lock stolen.
- **No TTL / manual release only.** Rejected: a crashed holder would block recovery indefinitely.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Exactly one instance recovers any given negotiation — no concurrent state-write race | Up to ~10 minutes of failover latency if the lock holder crashes mid-recovery |
| A crashed holder self-heals on TTL expiry with no manual intervention | TTL too short would let a slow-but-healthy holder lose the lock mid-flight (the original 5-min TTL exhibited this) |

The TTL is a direct tension between failover latency and protection coverage: 600s is the chosen point, sized to exceed worst-case single-node execution (≈500s) with margin.

## Results

Exactly one instance recovers any given negotiation (enforced by the conditional write). A crashed holder self-heals on TTL expiry. Lock-acquisition failure, or a lock held past 600s without checkpoint progress, raises REQUIRES_ATTENTION trigger #7 (`recovery_lock_timeout`) so ops can intervene.

**Scope note 2026-08-04 (impl PR #257).** Until this PR the lock was correct but guarding a replay that could not occur: AD-14's checkpoint write had no callers, so `run_recovery_flow` never found a checkpoint to resume and the concurrent-replay race this lock exists to prevent was not reachable. The lock itself needed no change — it is keyed on `(tenant_id, negotiation_id)` and enforced by a conditional write, independently of what the recovery flow then does. Two consequences of the gap are worth recording: the "lock held past 600s **without checkpoint progress**" condition in the paragraph above had no checkpoint progress to observe, so trigger #7 could only ever fire on the acquisition-failure path; and the 600s TTL derivation (A2A timeout × retries + backoff + **checkpoint write budget** ≈ 500s) budgeted for a write that was not happening. Now that all six node Lambdas write checkpoints, both the sizing rationale and the progress-based condition describe the running system. Neither has been re-measured against live recovery.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
