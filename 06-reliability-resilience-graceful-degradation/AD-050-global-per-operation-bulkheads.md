# AD-050 — Global Per-Operation Bulkheads in v1.0; Per-Tenant Sharding Deferred

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-50 · **Source PRD:** PRD-006 · **Status:** Accepted (outbound mechanism removed as dead code, PR #109 — see Results) · **Related:** AD-79, AD-45

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

**Removed as dead code (PR #109, 2026-07-02).** REQ-R300–R302 are SUPERSEDED. The deployed graph shape turned out to be one A2A call per round per node with no orchestrator-side fan-out for the outbound bulkhead to bound — `orchestrator/resilience/bulkhead.py` and its semaphores were unreachable in practice, never queuing and never emitting `bulkhead.queue`. The module and its `bulkhead_config` seed block were deleted; `__init__.py`, `config.py`, and `admission.py` docstrings were updated to drop the stale references. Live backpressure on shared resources is the inbound admission control (§2.4.1 / AD-79) alone — there is no separate per-operation-type cap today.

The Phase 2 per-tenant-sharding plan described above is moot: there is no v1.0 outbound bulkhead left to shard. If per-operation backpressure is needed in the future (e.g., a fan-out shape reappears), it should be re-derived against the shape that actually exists rather than resurrected from this design.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
