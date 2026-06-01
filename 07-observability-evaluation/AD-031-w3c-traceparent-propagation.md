# AD-031 — W3C `traceparent` Propagation Across All A2A Calls

**Theme:** Observability & Evaluation  **Catalog:** AD-31 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-29, AD-13, AD-30

## Context

A single negotiation runs across a 7-node graph and multiple agents that communicate over A2A (JSON-RPC 2.0). Without a shared correlation context, each agent and each hop produces its own disconnected trace, making it impossible to reconstruct an end-to-end negotiation view or to give evaluators and auditors the complete picture they need. Resilience events (retries, circuit-breaker transitions, recovery events) would also be invisible within the operation context where they occur.

## Decision

Propagate the W3C `traceparent` header on every A2A invocation so that the entire negotiation surfaces as one distributed trace. Resilience operations — retries, circuit-breaker transitions, idempotency cache hits, and recovery events — are recorded as span events within the relevant operation span rather than as separate disconnected traces. Propagation is enforced through the shared `invoke_agent_runtime` wrapper (AD-13) rather than at individual call sites.

## Alternatives Considered

- **Homegrown correlation ID.** Rejected: a proprietary header provides the same correlation function but loses interoperability with OTEL tooling and downstream consumers that speak the W3C standard.
- **Status quo / per-agent disconnected traces.** Rejected: disconnected traces make end-to-end negotiation reconstruction and evaluator input impossible, defeating the observability architecture (AD-29) and the evaluation prerequisite (AD-30).

## Trade-offs

| Gained | Given up |
| --- | --- |
| One coherent distributed trace per negotiation, consumable by evaluators via AD-30's Transaction Search and by the per-tenant audit-trail export (REQ-O212) | Strict adherence to the W3C `traceparent` format is required across every A2A hop |
| Standard OTEL tooling interoperability for all downstream consumers | The whole-negotiation trace is only as complete as its weakest hop; a single call that fails to propagate the header breaks the chain |
| Enforcement in one place (AD-13 wrapper) rather than at every call site | Creates a hard dependency on the shared invocation wrapper as the sole propagation enforcement point |

## Results

One coherent distributed trace per negotiation is the substrate that evaluators consume (via AD-30's Transaction Search) and the basis for the per-tenant audit-trail export in REQ-O212. Propagation correctness depends entirely on the shared `invoke_agent_runtime` wrapper (AD-13); any bypass of that wrapper would silently break the trace chain. AD-30 must be enabled for evaluators to read the resulting traces.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
