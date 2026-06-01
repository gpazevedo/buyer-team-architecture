# AD-029 — Three-Layer Observability (Platform / Application / Domain)

**Theme:** Observability & Evaluation  **Catalog:** AD-29 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-30, AD-31, AD-13

## Context

Agentic systems resist conventional monitoring for four reasons: non-determinism, multi-hop reasoning, cost opacity, and silent quality drift. A single undifferentiated stream of metrics cannot serve the three audiences who care about the system — SRE, procurement, and FinOps — because the questions they ask are at different altitudes. Runtime health ("is the agent erroring?") and business outcome ("did we save money?") must not be coupled to the same emission path. Without separation, new domain metrics would couple to runtime internals and per-audience dashboards would become impossible to maintain without breaking the others.

## Decision

Observability is split into three independent layers, each with its own ownership and CloudWatch namespace:

- **Layer 1 — Platform:** AgentCore built-in metrics (session count, invocation latency p50/p90/p99, error count/rate, throttle count, token usage) under `AWS/BedrockAgentCore`, emitted with no code instrumentation required.
- **Layer 2 — Application:** ADOT/OTEL spans carrying procurement-specific and security attributes (`procurement.tenant_id`, `procurement.negotiation_id`, `security.steering.*`, etc.) via custom instrumentation.
- **Layer 3 — Domain:** Custom OTEL business metrics published as EMF to `procurement/business`, `procurement/kpi`, `procurement/cost`, `procurement/resilience`, `procurement/security`, and `procurement/tenant`.

## Alternatives Considered

- **Single unified observability layer.** Rejected: it would either lack business context (tenant, savings, Kraljic quadrant) or couple business KPIs directly to runtime internals, making per-audience dashboards unworkable.
- **Two layers (Platform + combined Application/Domain).** Rejected: merging application traces with domain EMF metrics conflates operational debugging with business KPI reporting, preventing independent evolution of either.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Clean per-audience dashboards (Operations, Business, Cost) that can each evolve independently | A single negotiation now spans all three layers, requiring a correlation mechanism (AD-31) to recombine |
| Layer 1 costs nothing to emit; no code instrumentation for platform metrics | Layers 2 and 3 require explicit instrumentation and maintained contracts per metric |
| New domain metrics can be added purely at Layer 3 without disturbing Layer 1 or 2 | Domain/cost metrics with high-cardinality dimensions (e.g., `negotiation_id`) are a known CloudWatch cost risk flagged as an open follow-up in PRD-004 |

## Results

Three role-aligned dashboards fall out naturally: Operations (SRE), Business (procurement), Cost (FinOps). Each layer can change without disturbing the others; new domain metrics were added over successive PRD-004 revisions purely at Layer 3. AD-31 (W3C `traceparent` propagation) provides the cross-layer correlation substrate; AD-30 (CloudWatch Transaction Search) is the prerequisite for evaluators to read Layer 2 traces. The accepted liability is CloudWatch cardinality and cost management on the domain layer, particularly for the unbounded-cardinality `negotiation_id` dimension.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
