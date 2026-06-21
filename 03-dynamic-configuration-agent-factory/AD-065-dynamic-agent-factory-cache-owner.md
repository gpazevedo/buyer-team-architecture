# AD-065 — DynamicAgentFactory Is the Single Request-Assembly + Cache-Checkpoint Owner

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-65 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-28, AD-59, AD-63, AD-25, AD-22, AD-23, AD-95

## Context

Prompt-cache hit rate (AD-59, ~90% input-token saving) depends on a byte-identical request prefix before the cache checkpoint. If any per-invocation or per-tenant value leaks before that checkpoint, the prefix fragments and cache writes multiply by the cardinality of the leaked dimension. Seven different agent authors assembling requests independently — each making a local decision about what goes before the checkpoint — cannot reliably maintain this invariant. It must be owned structurally in one place, not left to convention.

## Decision

`DynamicAgentFactory` is the single point where the Bedrock request is assembled and the sole owner of the cache-prefix invariant: only the `system` prompt and `tools` schema fall before the cache checkpoint; every per-invocation input — payload, round state, retrieved memory context, and resolved per-tenant thresholds — falls after it. The checkpoint is placed explicitly at the `tools`/message boundary rather than relying on default placement. The invariant is enforced by test (REQ-C004), not convention.

## Alternatives Considered

- **Each agent assembles its own request.** Rejected: distributes ownership of the cache-prefix invariant across seven agent authors, making silent regressions likely; demonstrated to produce drift (two agents corrected in v1.0.23).
- **Convention-only enforcement (code review).** Rejected: the v1.0.23 correction showed the rule silently rots without an automated test; convention is insufficient for a property that is invisible in normal operation.
- **Per-tenant system prompts with interpolated thresholds.** Rejected: fragments the cache per tenant (multiplies writes by tenant cardinality) and is a §1.1 violation — resolved thresholds belong in agent metadata for hooks, not in the prompt.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The ~90% cache saving is a tested property rather than an accident; the invariant is enforced in exactly one place | Agents may not interpolate resolved thresholds or tenant values into the `system` prompt, even when that is the "obvious" place to put them |
| One chokepoint for request assembly means the rule does not depend on seven authors making the same correct choice | Build-determining config changes (`model_id`, `temperature`, `max_tokens`, `prompt_caching_enabled`) define a new cache context; a re-warm dip is expected after any such change |
| The factory's structural position forces governance, filters, and guardrails into tools and hooks rather than prompts — a deliberate architectural ripple consistent with AD-22 and AD-23 | Request assembly flexibility is intentionally constrained at the factory boundary |

The constraint that resolved thresholds and per-tenant values cannot enter the `system` prompt is not a free choice — it is a direct consequence of the Tools-as-Boundaries (AD-22) and Steering-over-Prompting (AD-23) principles, which kept external data and guardrails out of the prompt for security reasons. AD-65 makes that property additionally load-bearing for cost.

## Results

Realized in PRD-010 §3.4: explicit checkpoint placement at the `tools`/message boundary, prefix-purity test REQ-C004, and the `DynamicAgentFactory` assembly contract. This is the structural counterpart to AD-28 (prefix-purity rule) — AD-65 specifies the single place where that rule is enforced. Together they turn the AD-59 (~90% cache saving) from an optimistic projection into a tested system property. Follow-on decisions that depend on this chokepoint: AD-28 (purity invariant), AD-59 (cost lever), and AD-63 (read-once config feeding the factory).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
