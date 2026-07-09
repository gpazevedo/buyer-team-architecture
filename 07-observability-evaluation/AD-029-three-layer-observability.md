# AD-029 — Three-Layer Observability (Platform / Application / Domain)

**Theme:** Observability & Evaluation  **Catalog:** AD-29 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-30, AD-31, AD-13, AD-115, AD-120, AD-121

## Context

Agentic systems resist conventional monitoring for four reasons: non-determinism, multi-hop reasoning, cost opacity, and silent quality drift. A single undifferentiated stream of metrics cannot serve the three audiences who care about the system — SRE, procurement, and FinOps — because the questions they ask are at different altitudes. Runtime health ("is the agent erroring?") and business outcome ("did we save money?") must not be coupled to the same emission path. Without separation, new domain metrics would couple to runtime internals and per-audience dashboards would become impossible to maintain without breaking the others.

## Decision

Observability is split into three independent layers, each with its own ownership and CloudWatch namespace:

- **Layer 1 — Platform:** AgentCore built-in metrics (session count, invocation latency, error count/rate, throttle count, token usage) under `AWS/BedrockAgentCore`, emitted with no code instrumentation required.
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

**Update 2026-07-04/05 — Layer 1 real namespace corrected; Layer 1 alarms and new Layer 3 metrics shipped.** The Layer 1 namespace is `AWS/Bedrock-AgentCore` (hyphenated) with real metric names `Errors` and `Latency` (also `SystemErrors`/`UserErrors`/`Duration`/`Sessions`/`Invocations`), confirmed against live dev CloudWatch data via `list_metrics` — not the unhyphenated `AWS/BedrockAgentCore` name this ADR originally stated and PRD-004 §3.3 guessed. `infra/agent_runtime_alarms.tf` (PR #139) adds 16 `aws_cloudwatch_metric_alarm` resources (Errors + Latency, `for_each` over the 7 agent runtimes + the skill runtime) on this corrected namespace, routed to the same `evaluation_alerts` SNS topic AD-34's alarms use — Layer 1 was previously "no code instrumentation required" but also **no alarms**; an agent up-but-erroring or slow was invisible until it caused a node Lambda failure. The Latency threshold (60s) is a first-pass guess, not a tuned SLO — there is no fixed per-invocation timeout for agent runtimes the way Lambda has one.

New Layer 3 domain metrics shipped in the same pass (PR #140): `tokens.input`/`tokens.output` under `procurement/cost` (dimensioned by `agent_name` + `model_tier`, emitted from `buyer_agent_core.observability.log_usage()`), `kraljic.classification_source` under `procurement/business` (one `emit_metric` call at the single `_response()` choke point all 5 Kraljic classification paths return through), and `governance.violation_count` under `procurement/business` (emitted from each of the 6 agents' `steering.py` guards at their block/intervention point, dimensioned by `violation_type`). A role-aligned Business dashboard (`infra/modules/observability/dashboard.tf`, PR #137/138) now visualizes negotiation lifecycle, bid/governance/approval, savings, and the new token-usage widgets — none of these three new metrics has an alarm yet; thresholds need real dev data first. `emit_metric` itself gained a self-observing failure mode (AD-115) as part of this same body of work.

**Update 2026-07-08 — Layer 3 emission corrected to true EMF; approval-gate trace-continuity gap closed.** The Layer 3 decision above always said "published as EMF," but until PR #164 (orchestrator `resilience/metrics.py`) and PR #166 (agent-side `buyer_agent_core/metrics.py`), `emit_metric` actually called a synchronous `boto3 put_metric_data` — a 5s-connect + 5s-read-timeout network call, retried once, sitting in the A2A hot path (`agent_invocation.py`'s `finally` block, every per-node emission, every LLM step's `log_usage`). Both PRs replaced it with a genuine EMF stdout log line, removing up to ~20s of blocking I/O per call in the failure case and real network latency even on the happy path — the implementation now matches what this ADR described from the start. This also retired AD-115's `put_metric_data`-failure fallback datapoint, since there is no longer a remote call for the primary path to fail (see AD-115's correction). Separately, PR #164 closed the last hole in AD-31's promised single coherent trace: the human-approval-gate pause (`node_approval_gate`'s `waitForTaskToken`) previously broke trace continuity on *every* task-token resume, not only human-reviewed ones, because `_send_success` shipped its output to Step Functions before the (then post-hoc) `inject_trace_context` call could run. See AD-31's update and the new **AD-120** for how `correlation_id` and the OTEL carrier now survive that pause as persisted row data rather than in-band propagation.

**Update 2026-07-09 — Dashboards re-split along the layer model; meta-observability heartbeat added to Platform.** The three role-aligned dashboards described in the original Results section above (Operations/SRE, Business/procurement, Cost/FinOps) have been replaced by four, realigned directly to this ADR's own Layer 1/2/3 split rather than to audience alone: **Platform** (pure `AWS/*` infra health, Layer 1), **Application** (circuit breakers and eval-quality signal, plus 5 newly-added `procurement/resilience` widgets — compensation/saga outcomes, award-retraction/bid-withdrawal notices, DLQ redrive outcomes, recovery-lock health, and the two REQUIRES_ATTENTION escalation triggers — found via a full-repo audit of `emit_metric(` call sites that had never had a dashboard widget, Layer 2), **Domain** (renamed from Business; `procurement/business`/`procurement/kpi`, Layer 3), and **FinOps** (cost, pre-existing, now the sole home for the Token Usage widget dropped from Domain). Platform also plots **AD-121**'s `pipeline_heartbeat` metric alongside the pure `AWS/*` widgets — a custom `procurement/observability` metric, but a platform-health question ("is the metrics pipeline itself alive") rather than a business one.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
