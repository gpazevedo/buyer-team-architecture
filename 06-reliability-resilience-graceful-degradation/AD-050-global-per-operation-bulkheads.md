# AD-050 — Global Per-Operation Bulkheads in v1.0; Per-Tenant Sharding Deferred

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-50 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-79, AD-45

## Context

Concurrency must be bounded per operation type — spot bid invitations, A2A calls, DynamoDB writes, supplier MCP — so one slow or failing dependency cannot exhaust shared resources and starve other negotiations. AgentCore runs a microVM per session, so there are no Runtime instances to scale horizontally; bulkheads are the primary backpressure mechanism. Without per-operation limits, a single large auction could saturate the spot-bid pool, blocking all concurrent negotiations system-wide.

## Decision

Ship global per-operation-type bulkheads in v1.0 with caps sourced from `{env}-system-config`: spot bid invitations 200 global / 20 per-auction, A2A agent calls 10, DynamoDB writes 50, supplier MCP 20. Defer per-tenant sharding within these bulkheads to Phase 2. All parameters are tunable from config without a redeploy (AD-45).

## Alternatives Considered

- **Per-tenant bulkhead sharding from v1.0.** Rejected: significantly more complex to implement and operate; the v1.0 admission control (AD-79) already provides per-tenant fairness at the negotiation-admission surface, which is the most important fairness boundary.
- **No explicit bulkheads; rely on AgentCore session limits.** Rejected: AgentCore microVM limits apply per session, not per operation type; one auction's invitation fan-out would exhaust shared operation-type capacity without a separate cap.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Simple, immediately effective backpressure on shared resources | Per-tenant fairness within a single operation type — a tenant running ten or more concurrent auctions can still saturate the global spot-bid pool |
| The per-auction sub-cap (20 of 200) prevents a single auction monopolizing the spot-bid pool | A known starvation risk explicitly accepted for v1.0 and deferred to Phase 2 (PRD-016 §8) |
| A2A and DDB-write bulkheads remain global indefinitely (graph-topology-bounded; DynamoDB self-limits via throttling) | Phase 2 per-tenant sharding adds operational complexity when it ships |

Fairness at the negotiation-admission surface is already provided in v1.0 by AD-79 (reserved + max + weight model at Node 1); only the operation-level bulkhead sharding within an operation type is deferred.

## Results

Global bulkhead caps are enforced per REQ-R300/R302. On semaphore exhaustion the request is queued and a `bulkhead.queue` metric is emitted. The Phase 2 plan reuses the same reserved+max model as admission control (AD-79), starting with spot-bid invitations (the most monopolizable), then supplier MCP. A2A and DynamoDB-write bulkheads are not candidates for per-tenant sharding because their caps are bounded by graph topology and DynamoDB's own throttling respectively.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
