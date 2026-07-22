# AD-129 — DynamoDB-Only Substitute for Mem0 IP-1, Isolated from the Agent-Failure Path

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-129 · **Source PRD:** PRD-014 · **Status:** Accepted · **Related:** AD-72, AD-74, AD-46, AD-3, AD-38

## Context

AD-72 and AD-74 designed supplier relationship history (IP-1) to run on Mem0. By 2026-07-22, Mem0 was still entirely unwired in dev — no SaaS account exists, and the agents have no NAT/VPC egress to reach it — while `strategic_partnership`'s relationship-history block continued deriving `past_negotiations` and `performance_trend` from a stable hash of the supplier ID, never from a real outcome. AD-72 explicitly considered and rejected "store cross-negotiation facts in bespoke DynamoDB tables" as an alternative to Mem0, on the grounds that it is heavier to build/operate and lacks Mem0's semantic search. That trade-off holds for the full four-IP design, but IP-1's actual need — a per-supplier win/loss tally and win-rate, not semantic retrieval over free-text history — does not require semantic search at all, and Mem0's account/networking prerequisites had no delivery date.

## Decision

Ship IP-1 now as a real, non-Mem0 substitute rather than continue deriving it from a hash. A new `supplier-memory` DynamoDB table (`tenant_id` hash key / `sk = "{supplier_id}#{negotiation_id}"` range key, no TTL) records each candidate supplier's outcome (`AWARDED` / `NOT_AWARDED`) and the real negotiated TCO after every successful strategic-partnership negotiation. `_query_supplier_memory` reads it back inside `_precompute_supplier_analysis`, which runs *before* the strategic agent is invoked: a supplier with recorded history gets a real `past_negotiations` count and win-rate-derived `performance_trend`; a supplier with none yet (cold start) falls back byte-for-byte to the existing hash-derived values, so there is no behavior change until real history accumulates. The read and the write are each wrapped in their own try/except, deliberately kept outside `_strategic_partnership`'s broad agent-invocation-failure handler — a supplier-memory hiccup (table not yet created, a transient throttle) must degrade to the cold-start values, not be mistaken for "the agent failed" and trigger deterministic fallback re-pricing or a wrongful escalation to `REQUIRES_ATTENTION` for a single/limited-source supplier. This is scoped to IP-1 only; IP-2 (bid patterns), IP-3 (approver preferences), and IP-4 (communication effectiveness) remain unimplemented, with no call sites anywhere in the agents.

## Alternatives Considered

- **Wait for Mem0 to be wired, keep IP-1 hash-derived until then.** Rejected: the hash-derived values were never a real signal, and Mem0's account/NAT prerequisites had no committed timeline — IP-1's decision-quality gap (AD-74's stated highest-impact IP) would stay open indefinitely for no infrastructure reason specific to IP-1 itself.
- **Stand up Mem0 now just to unblock IP-1.** Rejected: pulls forward an external SaaS dependency, a NAT gateway or VPC endpoint for outbound HTTPS, and the Mem0 circuit breaker (AD-74) for a feature that only needs a per-supplier tally — disproportionate infrastructure for the actual requirement.
- **Fold the write/read into `_strategic_partnership`'s existing failure handling.** Rejected: that handler's job is "the agent failed, apply deterministic fallback pricing or escalate" — a memory read failure is a different, much lower-severity event, and conflating them would turn a cold-start read miss into fallback pricing or a false `REQUIRES_ATTENTION` escalation on suppliers that never actually failed to negotiate.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A real, live decision-quality signal for IP-1 today, without waiting on Mem0's SaaS account or networking | No semantic search — this only answers "how has this supplier done," not free-text queries over negotiation history the way Mem0 eventually would |
| Isolation strength at least as strong as Mem0's design: `tenant_id`-partition, IAM-enforceable, matching the pattern used everywhere else in the system (AD-38) — strictly stronger than Mem0's filter-only `user_id` scoping (AD-73) | A second, parallel storage pattern for "cross-negotiation memory" alongside the Mem0 design AD-72/AD-74 still describe for IP-2 through IP-4 — two mental models for what is conceptually one capability until Mem0 lands |
| Zero incremental networking cost — rides the existing free DynamoDB Gateway VPC endpoint instead of requiring NAT egress | If/when Mem0 is eventually wired for IP-2–4, IP-1 either stays a permanent exception or needs a second migration onto Mem0 to unify the model |
| Failure isolation from `_strategic_partnership`'s agent-failure handler means a memory-layer bug cannot masquerade as a negotiation failure | One more failure-mode/observability surface (`supplier_memory.query_error` / `.write_error`) that must be watched independently of the agent's own decision logs |

## Results

Realized in `orchestrator/node_strategy_execute.py` (PR #236, 2026-07-22): `_query_supplier_memory`, `_record_supplier_memory`, and the `supplier-memory` table in `infra/modules/dynamodb/main.tf`. 372 orchestrator tests passing (+7 new), covering the write-path happy path, cold-start/real-history mixing within one negotiation, and the key regression — a full STRATEGIC run survives a totally broken supplier-memory table with the real agent-negotiated bid intact and zero `strategy.fallback_engaged` fires. `terraform validate` and a `-target`-scoped plan are clean; the live `dev-supplier-memory` table has not yet been created via `terraform apply`. AD-72 and AD-74 are updated to point here rather than describe IP-1 as Mem0-backed; IP-2 through IP-4 still await both Mem0 itself and a decision on whether they follow this same substitute pattern or wait for the SaaS account.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
