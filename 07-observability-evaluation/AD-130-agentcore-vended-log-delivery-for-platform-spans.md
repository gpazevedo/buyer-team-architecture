# AD-130 — CloudWatch Vended Log Delivery Required to Resolve AgentCore's Own Platform-Generated Spans

**Theme:** Observability & Evaluation  **Catalog:** AD-130 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-29, AD-30, AD-31, AD-52

## Context

AD-31 propagates W3C `traceparent` on every A2A call so a negotiation surfaces as one distributed trace. That covers everything our own code emits — the orchestrator's `agentcore.invoke` span and the agent container's application spans (via the explicit `message.metadata` carrier, since AgentCore does not forward arbitrary HTTP headers into the container). It does not cover a third participant: AgentCore Runtime itself. AgentCore Runtime is trace-aware and generates its own per-invocation span server-side, at the data-plane ingress that terminates `InvokeAgentRuntime`, upstream of the container. That span's ID is what the container's outermost ASGI span (`POST /`) extracts as its `Parent` — correctly, per the propagation contract — but the span itself is a separate, AWS-owned telemetry object, exported through neither the application's own ADOT pipeline (AD-29's Application layer) nor Lambda's native X-Ray facade trace (a distinct trace root generated because `TracingConfig.Mode=Active`, unrelated to either).

Discovered 2026-07-22 chasing a residual ~1% orphaned-parent rate left after bumping the agent-base image's `aws-opentelemetry-distro` floor from 0.10.0 to 0.18.0 (fixing a much larger ~35-37% orphan rate — that version stopped a documented ADOT bug that silently discarded inbound X-Ray/W3C context). Proven, not guessed: a `before-send` botocore hook on the orchestrator's `InvokeAgentRuntime` call captured the literal outgoing `X-Amzn-Trace-Id` header and confirmed Root and Parent both matched our own `agentcore.invoke` span exactly — ruling out every candidate on our side of the wire. The resulting agent-side `POST /` span still had a parent ID matching none of: our own OTel trace, Lambda's native X-Ray facade trace, or Transaction Search's `aws/spans` (AD-30). The ID was real (AgentCore's own service span, confirmed by AWS) but simply never delivered into the account.

## Decision

Enable CloudWatch vended log delivery (`logs:PutDeliverySource` with `logType=TRACES`, resource ARN of the AgentCore resource → `PutDeliveryDestination` type `XRAY` → `CreateDelivery` linking them) for every AgentCore Runtime and Gateway resource, so the platform's own per-invocation spans land in the account's X-Ray/Transaction Search store alongside the application-level spans AD-30/AD-31 already cover. Neither `agent_runtime` nor `gateway` model an observability field on Create/Update in either Terraform provider (the same AD-52 provider-gap shape), so this is wired via a boto3 `terraform_data` provisioner (`scripts/manage_runtime_observability.py`, mirroring the pre-existing `scripts/manage_gateway_observability.py` used for the receiving Gateway) rather than a native resource.

## Alternatives Considered

- **Accept the dangling parent as cosmetic noise.** Rejected: it isn't cosmetic — it's the one hop in the architecture (AgentCore's own ingress) that was a complete black box, undermining exactly the "one coherent distributed trace per negotiation" property AD-31 exists to guarantee.
- **Suppress the ASGI auto-instrumentor (`OTEL_PYTHON_DISABLED_INSTRUMENTATIONS`) so no span exists to hold the unresolvable reference.** Rejected: deletes the dangler instead of resolving it, at the cost of losing the container's own request-level timing span. Strictly worse than making the real span resolvable.
- **Override `OTEL_PROPAGATORS` in the container to ignore the platform-injected header.** Rejected: severs `Root` continuity — the container subtree would start a disjoint trace instead of continuing the orchestrator's, which is the exact failure mode AD-31 was written to prevent.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Complete trace lineage — `agentcore.invoke` → AgentCore Runtime's own service span (`origin: AWS::BedrockAgentCore::Runtime`) → `POST /` → application spans — with no unresolvable hop | Another AD-52-shaped provider gap: CloudWatch Delivery resources exist outside Terraform state, tracked only by the provisioner's own idempotent-create behavior |
| Visibility into AgentCore's own ingress latency/behavior, previously fully opaque | One more `terraform_data` provisioner per resource (12 total: 6 runtimes here, 1 pre-existing on the Gateway) that must stay in sync with resource replacement via `triggers_replace` |
| Same mechanism already proven on the receiving Gateway (Cedar policy observability) — no new pattern, just its correct generalization to Runtimes | Requires AD-30's Transaction Search prerequisite already in force; adds no new account-level toggle but inherits that one's failure mode |

## Results

Live-verified on `dev_kraljic_classifier` (2026-07-22): before the delivery, ~1% of live traces had a dangling parent at the `POST /` hop; after, the identical invocation path produced a fully-resolved trace, the gap span landing with `origin: AWS::BedrockAgentCore::Runtime`, `span.name: AgentCore.Runtime.Invoke`. Formalized in impl PR #249 for all 6 agent runtimes (`kraljic-classifier`, `spot-bidding`, `leverage-auction`, `bottleneck-negotiation`, `strategic-partnership`, `award-comms`); the receiving Gateway already had the same delivery configured from earlier Cedar Policy observability work (AD-52's provisioner pattern), but — until this ADR — undocumented as a decision in its own right rather than an implementation detail of that work. `APPLICATION_LOGS` delivery is deliberately not part of this decision for Runtimes: container logs already land in their own native `/aws/bedrock-agentcore/runtimes/...` log group without a delivery; only `TRACES` was missing.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
