# AD-013 — Shared `invoke_agent_runtime` Wrapper

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-13 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-4, AD-45, AD-20, AD-31, AD-61

## Context

Every agent node (2, 4a–4d, 5, 7) makes an A2A call into a Bedrock AgentCore Runtime. Resilience patterns (retry, circuit breaker, bulkhead, timeout) and per-tenant cost attribution must be applied uniformly; if each node called `bedrock-agentcore:InvokeAgentRuntime` directly, those concerns would be re-implemented per node and drift apart. W3C `traceparent` propagation (AD-31) and the `memory_degraded` signal (AD-20) also need a single, disciplined read/write site.

## Decision

Mandate that every agent-node executor invoke its agent exclusively through the shared `invoke_agent_runtime` wrapper (PRD-006 §2.7), never the raw `InvokeAgentRuntime` API (REQ-G008). The wrapper applies all PRD-006 §2.1–§2.5 resilience patterns, propagates W3C `traceparent` across A2A boundaries, emits the per-tenant `agentcore.session_seconds` cost signal (consumed by PRD-009 FinOps), and returns the `memory_degraded` field that the orchestrator reads after each call.

## Alternatives Considered

- **Direct `InvokeAgentRuntime` per node.** Rejected: duplicated resilience logic, inconsistent cost emission, and no single chokepoint for policy or trace propagation.
- **Per-node bespoke wrappers.** Rejected: the same drift problem in a thinner disguise — resilience and attribution behaviour would still diverge across nodes.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Uniform resilience, cost attribution, and trace propagation across all six agent nodes | A shared critical-path dependency — a wrapper regression affects all six agents simultaneously |
| `memory_degraded` and `agentcore.session_seconds` have a single, testable read/write site | One additional layer of indirection on every A2A call |
| Tuning (retry counts, breaker thresholds, timeouts) is config-driven without touching node code (AD-45) | All callers must route through the wrapper; direct SDK use is prohibited |

The concentration of failure risk is accepted because it is exactly what makes uniform behaviour and cost attribution enforceable rather than conventional.

## Results

Retry, circuit-breaking, bulkheads, and timeouts behave identically across all agent nodes and are tunable from `{env}-system-config` without touching node code (AD-45). Per-tenant AgentCore Runtime cost is attributable because `agentcore.session_seconds` is emitted from one measured point, feeding the PRD-009 FinOps pipeline (AD-61). The `memory_degraded` signal (AD-20) rides back on the same response, giving degradation handling a single read site. W3C `traceparent` propagation (AD-31) is enforced at the wrapper, making trace completeness a structural guarantee rather than a convention.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
