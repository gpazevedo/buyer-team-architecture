# AD-028 — Prompt-Cache Prefix Purity

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-28 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-22, AD-23, AD-65, AD-59, AD-25

## Context

Bedrock prompt caching is the primary cost lever — roughly 90% input-token savings, supporting the `<$0.10/negotiation` target (PRD-009 §3). The saving materializes only if the request prefix before the cache checkpoint is byte-identical across invocations. Any per-invocation value that leaks into the prefix — supplier data, round state, resolved thresholds, memory context — differs every call, forcing a cache write at full input-token cost instead of a read. Two agents' caching rows had drifted from this rule in v1.0.22 and were corrected in v1.0.23.

## Decision

For every agent the cached prefix is exactly the invocation-invariant content — the `system` prompt plus the `tools` schema block — and nothing else. All per-invocation content sits after the cache checkpoint: the category/supplier/bid payload, round number and prior-round state, retrieved memory context, and resolved per-tenant thresholds. `DynamicAgentFactory` is the single assembly point and enforces this invariant with a prefix-purity test (REQ-A707, REQ-C004). This is a consequence of AD-22 (Tools-as-Boundaries) and AD-23 (Steering-over-Prompting): because data and guardrails are kept out of the system prompt by those principles, the `system` block is already invocation-invariant.

## Alternatives Considered

- **Per-tenant system prompts with interpolated thresholds.** Rejected: fragments the cache per tenant, multiplying cache writes by tenant cardinality and eliminating cross-invocation prefix sharing.
- **Governance rules in the system prompt.** Rejected: makes the prefix vary whenever thresholds are tuned; also a §1.1 (AD-22/AD-23) violation; steering hooks are the correct location.
- **Memory context in the system prompt.** Rejected: per-negotiation and per-supplier by nature; placing it before the checkpoint causes a cache miss on every invocation.
- **Convention-only enforcement.** Rejected: demonstrated to silently rot — two agents drifted before the v1.0.23 correction; the invariant must be backed by an automated test (REQ-C004).

## Trade-offs

| Gained | Given up |
| --- | --- |
| The primary cost lever actually fires — high cache-hit rates hold the `<$0.10/negotiation` cost target | A permanent constraint on every agent author: nothing per-invocation may enter the `system` block |
| Forces clean separation of data from prompt, reinforcing AD-22 and AD-23 as jointly load-bearing for cost as well as security | The regression is silent and easy to introduce; it must be caught by automated test, not code review |
| Backed by a prefix-purity test (REQ-C004), so the invariant cannot quietly rot | Cache writes carry a ~2× premium at 1h TTL; the saving is on reads, not writes |

Cache writes at the 1h TTL carry a ~2× per-token premium. This is acceptable because the write cost is fixed per build and the read saving (~90%) applies across every subsequent invocation within the TTL window — the economics are net-positive at any realistic invocation rate.

## Results

Realized in `DynamicAgentFactory` (PRD-010 §3.4, REQ-C004) as an explicit checkpoint placed at the `tools`/message boundary and enforced by a prefix-purity test. The AD-22/AD-23 principles are now jointly load-bearing for cost: any future change that threads supplier data, round state, governance values, or resolved tenant thresholds into the `system` prompt is simultaneously a prompt-cache regression and a §1.1 violation, caught by the REQ-C004 test. AD-65 makes the factory the sole assembly point so the rule is enforced in one place, not by convention across seven agent authors.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
