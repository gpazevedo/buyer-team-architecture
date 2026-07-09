# AD-059 — Prompt Caching for All 7 Agents

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-59 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-28, AD-58, AD-65, AD-57

## Context

Each agent's system prompt and tool schemas are large (hundreds to thousands of tokens) and byte-identical across invocations for the same agent configuration. Re-sending them at full input-token price on every call is the dominant per-invocation input cost. Bedrock prompt caching stores the request prefix — system prompt and tool schemas — so repeat invocations that present a byte-identical prefix pay only ~10% of the standard input rate on the cached portion. Without this decision, prompt caching must be explicitly enabled per agent and the invariant that makes it effective (prefix must remain byte-identical across calls) must be informally enforced. A realized ~90% input-token saving is the primary mechanism by which the worked example drops from ~$0.095 to ~$0.061 and the system-wide target of <$0.10/negotiation (PRD-001 §2.2 KR3.2) is met.

## Decision

Enable Bedrock prompt caching for all 7 agents, caching the invocation-invariant request prefix (system prompt + tool schemas) at the maximum TTL supported by the configured model: 5 minutes for Nova, the only deployed model family (AD-58's "Scenario A/B" framing for this choice was never realized — see AD-58's correction note). A cache write carries a premium over standard input (~1.25× at the 5-minute TTL), so an agent that never lands a second read before its prefix expires can cost marginally more than uncached. The prefix invariant — every per-invocation input falls after the cache checkpoint — is owned and tested by `DynamicAgentFactory` (AD-65, PRD-010 §3.4 / REQ-C004; agent-side: AD-28, PRD-003 §3.4 / REQ-A707).

## Alternatives Considered

- **Selective caching (only high-cadence agents).** Rejected: misses the saving on lower-cadence agents that still benefit, and creates inconsistent assembly logic in `DynamicAgentFactory`; the write-premium risk on sparse agents is monitored and acceptable.
- **No prompt caching.** Rejected: without the ~90% input-token saving the <$0.10/negotiation target is not achievable at the token volumes driven by the deployed agent set; prompt caching is the primary cost lever.
- **Manual per-agent cache configuration (not factory-owned).** Rejected: the byte-identical-prefix invariant cannot be reliably enforced without a single assembly owner; cache hits are silently foregone when any caller assembles a request differently.

## Trade-offs

| Gained | Given up |
| --- | --- |
| ~90% off input tokens on a cache read — the primary cost lever; transparent to application code above the factory | Cache writes carry a ~1.25× premium at the 5-minute TTL; an agent whose prefix expires before any second read costs marginally more than uncached |
| The saving is a tested property (REQ-C004 prefix-purity test), not an accident of convention | The realized saving is entirely contingent on a byte-identical prefix, which constrains how requests may be assembled; per-invocation data must never precede the cache checkpoint |
| Hit rates are monitored (`bedrock_prompt.cache_hit_rate`, <50%/15min alarm) to distinguish sparsity from churn | Nova's 5-minute TTL means sparsely-invoked agents have materially lower hit rates; TTL choice is a hard dependency |

Sub-target agents are limited by invocation cadence against the 5-minute TTL, not by prefix churn — prefix stability removes churn as a cause. A model-config change (`model_id`, `temperature`, `max_tokens`, `prompt_caching_enabled`) defines a new cache context by design; the first post-change invocation is necessarily a cache write and hit rate dips transiently until the new prefix re-warms. (The illustrative per-agent hit-rate estimates this ADR originally gave assumed a 1-hour Claude TTL that was never deployed — see the 2026-07-08 update below for what's actually true of the Nova-only fleet.)

## Results

Prompt caching is enabled for all agents via system-config `features.prompt_caching_enabled` (PRD-010 §2.3). The cache-prefix invariant is enforced in `DynamicAgentFactory` (AD-65) and tested by REQ-C004 / REQ-A707. Monitored via `bedrock_prompt.cache_hit_rate` in `procurement/operations`; per-agent `token.cache_write_count` / `token.cache_read_count` (dimensioned by `agent_name`) distinguish sparsity (sustained high write:read ratio) from churn (writes correlated with config changes). The <$0.10/negotiation target (PRD-009 §2.3) depends on this saving being realized. There is only one TTL (Nova's 5 minutes) in any deployed environment — see the update below for how much of the prefix that TTL actually covers.

**Update 2026-07-08 — Nova's cache coverage was narrower than "system prompt + tool schemas" until PR #167.** Live boto3 Converse testing against Nova micro/lite/pro found the request-prefix caching this ADR describes was only half-true for Scenario B: `toolConfig.tools` rejects a `cachePoint` on Nova (`ValidationException`, confirmed unchanged), so only the system-prompt portion of the prefix was ever eligible for a cache write — the tool-schema portion re-sent at full price on every call regardless of `PROMPT_CACHE_ENABLED`. `cache.py` now gates the two independently: `_model_supports_system_cache` (Claude + Nova) feeds `cached_system_prompt`; `_model_supports_tool_cache` (Claude only) feeds `model_cache_kwargs`. Because 5 of the 6 live dev agents run on Nova, this was the majority of the deployed fleet getting a materially smaller saving than "Prompt Caching for All 7 Agents" implies. Llama-based configurations get no caching benefit at all — `AccessDeniedException` on any `cachePoint` — which this ADR's "all 7 agents" framing does not distinguish from the Claude/Nova cases; a Llama-tiered agent silently pays full price with no cache-related error surfaced anywhere the fleet currently monitors (`bedrock_prompt.cache_hit_rate` would simply read near-zero, indistinguishable from the sparsity/churn cases this ADR already anticipates).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
