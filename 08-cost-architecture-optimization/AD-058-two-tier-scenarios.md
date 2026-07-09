# AD-058 — Two Supported Tier Scenarios (A: Claude, B: Nova)

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-58 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-57, AD-59, AD-25, AD-65

> **Correction (2026-07-08):** this decision was never realized. There is no `{env}-system-config` scenario switch and no code path that resolves to a Claude model — `packages/buyer_agent_core/buyer_agent_core/seed_mirror.py`'s `SEED_TIERS` hardcodes a single Amazon Nova (+ unused Llama `EvalLLM`) mapping, and `model_resolver.py` has no concept of a "scenario." "Scenario A" (Claude) was a planning-time option that was written up as if deployed but was in fact never built. The body below is preserved as the original (aspirational) decision record; treat every "Scenario A" and "operators can choose" statement as never having been true in any deployed environment. See PRD-009 v2.0.0 §2 for the real, single tier mapping.

## Context

The SimpleLLM / DefaultLLM tier labels introduced by AD-57 must resolve to concrete Bedrock model IDs. The choice of model family is not free: a steeper per-token discount (Nova) comes with a shorter prompt-cache TTL (5 minutes vs 1 hour for Claude 4.5), which changes the caching economics for sparsely-invoked agents (Bid Evaluation, quadrant agents) whose prefixes may expire between calls. Two credible mappings exist, each surfacing a different trade-off between per-token price and cache persistence. Hiding the choice as an undocumented constant would leave operators unable to reason about the cost impact of switching families.

## Decision

Support two explicit, documented scenarios as the only sanctioned tier mappings. Scenario A (Anthropic): DefaultLLM = Claude Sonnet 4.5 ($3/$15 per 1M in/out), SimpleLLM = Claude Haiku 4.5 ($1/$5), ~3× tier gap, 1-hour cache TTL, ~32K-token cache cap. Scenario B (Amazon Nova): DefaultLLM = Nova Pro ($0.80/$3.20), SimpleLLM = Nova Lite ($0.06/$0.24) or Micro ($0.035/$0.14), ~13–23× tier gap, 5-minute cache TTL, ~20K-token cache cap. Scenario A is the deployed mapping; both scenarios are specified with rates, TTLs, and cache caps in `{env}-system-config`.

## Alternatives Considered

- **Scenario A only (lock to Claude).** Rejected: forecloses a legitimate ~13–23× per-token cost reduction and removes the operator's ability to trade cache persistence for a lower absolute spend; the trade-off is worth documenting rather than hiding.
- **Scenario B only (lock to Nova).** Rejected: Nova's 5-minute cache TTL materially lowers hit rates for sparsely-invoked agents, and the wider tier discount is partly offset by more frequent cache misses; making this the only option removes the cache-persistence choice.
- **Unrestricted model selection (any model per agent).** Rejected: creates an unbounded space of tier combinations that cannot be systematically cost-modeled, alarmed, or supported operationally.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Operators can choose a far lower absolute per-token cost (Scenario B) or better cache persistence (Scenario A) with full visibility into what is traded | A single blessed model set; two scenarios must both be tested, documented, and supported |
| The discount-vs-cache-TTL interaction is explicit rather than hidden, preventing incorrect assumptions that "cheaper per token" equals "cheaper end-to-end" | Scenario B's wider tier discount is partially offset by lower cache hit rates on sparsely-invoked agents (5-minute window vs 1 hour), so the comparison requires per-agent cadence analysis |

The $0.061/negotiation figure in PRD-009 §2.2 uses Scenario A rates. Authoritative chargeable cost comes from CUR (AD-61), not the illustrative per-token constants in either scenario.

## Results

Both scenarios are specified in PRD-009 §2.0 with per-token rates, TTL, and cache cap. The scenario is selected via the `{env}-system-config` model-config group resolved by `DynamicAgentFactory` (AD-65). The TTL choice here is load-bearing for the prompt-cache hit-rate forecasts in AD-59 (sparsely-invoked agents under Scenario B's 5-minute TTL have materially lower hit rates). The CUR-authoritative cost pipeline (AD-61) captures the realized blended rate regardless of which scenario is active.

**None of the above "Results" happened.** There is no scenario-selection config group, and AD-61's CUR/Athena pipeline was never built (see its own correction note). What's real: one Nova+Llama tier mapping (`seed_mirror.py`), one 5-minute cache TTL (AD-59), and per-negotiation estimated cost from `resilience/pricing.py` accumulated into `negotiation.total_cost_usd` (PRD-009 v2.0.0 §6.1). Nothing distinguishes "which scenario is active" because there was only ever one.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
