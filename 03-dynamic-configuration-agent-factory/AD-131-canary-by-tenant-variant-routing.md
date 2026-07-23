# AD-131 — Tenant-Scoped Canary Rollout via Registry Variant Routing

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-131 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-63, AD-56, AD-66, AD-25, AD-16, AD-3

## Context

AD-56 established that AgentCore Runtimes have no fractional traffic routing, so the existing "canary" (PRD-007 §7.4) is a monitoring-only observation window over a full-fleet deploy — it validates a new image before declaring rollout complete, but it cannot answer "does this new model/prompt/config actually perform better for real tenant traffic" without exposing every tenant to it at once. That is a materially different question: a model swap or prompt rewrite is a behavioral change, not just a deployability check, and needs a way to expose it to a controlled subset of real tenants — first individually pinned, then a statistically meaningful population slice — while a shadow comparison proves safety before any tenant-visible traffic moves. `{env}-system-config`'s `registry` block (AD-63) already resolves `agents[logical].runtime_name` per agent but had no per-tenant override or population-split mechanism.

## Decision

Extend the registry resolver (`resolve_agent_runtime_name`, `graph_common.py`) with three additively-layered mechanisms, all keyed by `logical` agent name and `tenant_id`:

1. **Tenant pin** (`registry/tenant#<id>.agents[logical].variant`) — mirrors the existing `governance/tenant#<id>` overlay pattern. An operator pins one tenant to a named variant (e.g. `"canary"`); an undefined variant name raises `VariantNotFound` rather than silently falling back to the base runtime.
2. **Shadow testing** (`registry/tenant#<id>.agents[logical].shadow`) — the live-agent path fires an async, fire-and-forget self-invoke (`InvocationType="Event"`, same idiom as Node 7's `_eval_only`) that replays the same request against the variant runtime and records agreement in `{env}-agent-session-cache`, never affecting the already-returned classification. `evals/agentcore/shadow_diff.py` reports the agreement rate as the pre-promotion safety signal.
3. **Deterministic population split** (`registry.agents[logical].ab_split`, e.g. `{"canary": 0.1}`) — hashes `sha256(f"{logical}:{tenant_id}")` so a tenant's bucket assignment is stable across its own repeat requests, without needing an explicit per-tenant pin. `resolve_variant_label()` shares the same precedence walk and returns which variant actually served a request (`"primary"` / `"env_override"` / the variant name) for KPI tagging.

Precedence is env override > explicit tenant pin > `ab_split` > base runtime — a deliberate single-tenant override always wins over a population experiment. The mechanism is agent-agnostic by construction (built first against `kraljic_classifier`, then Terraform-generalized via `for_each` over the existing `aws_iam_role.agent_runtime` set to all 6 primary agents — `bid_evaluation` excluded, no canary runtime built for it). An operator-facing `scripts/promote_tenant_variant.py` reads the base registry to fail fast on an undefined variant, shallow-merges the tenant overlay (preserving sibling agents' pins/shadow flags), diffs before/after, and supports `--rollback`.

## Alternatives Considered

- **Wait for AgentCore to ship fractional traffic routing.** Rejected: no committed timeline (AD-56's stated constraint), and the actual need — validate a behavioral change against real tenant traffic before wide exposure — doesn't require traffic-layer routing at all if the application layer already resolves a runtime ARN per invocation.
- **A/B split only, no explicit tenant pin.** Rejected: an operator needs to expose a change to *specific*, chosen tenants (e.g. an internal or design-partner tenant) ahead of any population-level statistical rollout, not just a random sample.
- **Route via a separate routing service/proxy in front of AgentCore.** Rejected: `resolve_agent_runtime_name` already sits on every invocation path (AD-25's factory) and already reads `{env}-system-config`; a new routing tier would duplicate state and add a hop for no benefit over extending the existing resolver.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Real, tenant-traffic-validated canary rollout despite AgentCore having no fractional routing primitive | An operator-driven, config-based mechanism — no automatic canary-analysis/rollback like a service-mesh traffic-split would give |
| Fail-fast on misconfiguration (`VariantNotFound`) rather than silent fallback, at both the pin and the `ab_split` layer | A misconfigured variant now fails the negotiation to `REQUIRES_ATTENTION` (distinguished as `variant_misconfigured`, not `agent_unavailable`, since PR #245) rather than degrading gracefully to the base runtime |
| Shadow testing proves safety with zero tenant-visible risk before any real traffic moves | The self-invoke needs its own IAM grant per node Lambda (`InvokeSelfShadowOnly`, mirroring Node 7's `InvokeSelfEvalOnly`) — a gap that shipped inert until caught live (PR #244) |
| Stable per-tenant bucket assignment (hash-based) makes population experiments statistically legible across a tenant's own repeat traffic | A second "variant" concept alongside AD-56's deployment-time canary — two mechanisms both called "canary" that must not be conflated |

## Results

Realized across `orchestrator/graph_common.py` (`resolve_agent_runtime_name`, `resolve_variant_label`, `VariantNotFound`), `orchestrator/node_kraljic_classify.py` (shadow self-invoke), `infra/agent_runtimes.tf` (`for_each` canary runtimes for 6 agents), `scripts/promote_tenant_variant.py`, `scripts/set_ab_split.py`, `evals/agentcore/shadow_diff.py`, and a KPI-by-variant CloudWatch dashboard tagging `kraljic.classification_source` with the resolved label. Live-verified end-to-end on dev 2026-07-22 (tenant pin, cross-agent pin, live shadow comparison, population `ab_split` at 30% against 2000 synthetic tenants with zero stickiness mismatches over 200 repeats, and the fail-fast path). Two gaps found only by live exercise and fixed same-day: the registry had never been seeded with a `variants` block for any agent (config gap, not a code bug), and Node 2's shadow self-invoke had no IAM grant (PR #244). `bid_evaluation` remains without a canary runtime and is out of scope until one is built.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
