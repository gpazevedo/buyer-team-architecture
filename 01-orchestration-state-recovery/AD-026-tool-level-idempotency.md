# AD-026 — Tool-Level Idempotency via Dedup Keys / Session Cache

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-26 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-2, AD-22

## Context

AgentCore sessions are ephemeral and agents are re-invoked on two paths: crash recovery (Step Functions resumes from the last checkpoint) and governed cycle-back (Node 6 → Node 4x, after a human reject-with-retry). Without idempotency, recovery would re-send bid invitations, duplicate negotiation proposals, and re-score bids — producing duplicate supplier communications and, worse, ranking drift that changes the award recommendation.

## Decision

Every state-mutating tool implements idempotency and returns `status: "already_sent"` / `"already_calculated"` on a dedup hit. Communication tools dedup against `CommunicationLog` keyed by `(negotiation_id, supplier_id, type[, round_number])`. The compute tools `calculate_tco`, `assess_supplier_risk`, and `score_bid` use a DynamoDB-backed session cache (`{env}-agent-session-cache`, 72h TTL; `score_bid` keyed `score#{bid_id}` with an input-hash for change detection, REQ-A513). Pure reads (`collect_round_bids`, `get_market_benchmark`, etc.) are idempotent by nature.

## Alternatives Considered

- **No tool-level idempotency; rely solely on node-level dedup keys.** Rejected: node-level dedup prevents re-executing a completed node but cannot protect against duplicate tool calls within a node that crashes mid-execution.
- **In-memory session cache only (no DynamoDB backing).** Rejected: in-memory state is lost when the AgentCore session terminates; offers no protection across the session boundary that crash recovery crosses.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Crash recovery and cycle-back are safe — no duplicate emails, proposals, or purchase orders | Every state-mutating tool requires a designed dedup key and a lookup before acting, adding latency and DynamoDB cost |
| `score_bid` cache makes evaluation deterministic across restarts, stopping ranking drift (REQ-A513) | A 72h-TTL session cache table must be provisioned, monitored, and reasoned about |
| Operators can resume from checkpoint and reuse partial work after a DLQ escalation (REQ-A950b/c) | |

## Results

This is the decision that makes the ephemeral-sessions + long-lived Step Functions lifecycle model (AD-2) actually safe to recover. The DynamoDB-backed `score_bid` cache promoting recovery to deterministic was a specific audit win. The acknowledged residual is `generate_award_recommendation`, marked **⚠️ Partial**: re-invocation at temperature 0.1–0.3 can still produce a slightly different ranking. For BOTTLENECK/STRATEGIC the human gate at Node 6 absorbs it; for below-threshold NON_CRITICAL auto-award it is accepted as a v1.0 residual risk, with a deterministic tie-breaker deferred to v1.1.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
