# AD-057 — Model Tiering by Cognitive Demand

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-57 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-21, AD-58, AD-25, AD-65

## Context

Without a decision, every agent uses the strongest (and most expensive) model available. High-volume, templated work — Kraljic classification, spot bids, award notifications — does not require the reasoning power of the most capable model. A worst-case negotiation without any optimization costs $1.50–$3.00 (PRD-001 §2.2 KR3.2 sets a $0.10 average target). Model selection is the single largest structural lever available before caching, semantic caching, or batch inference are applied, because it reduces the per-token base rate on the agents that dominate token volume.

## Decision

Assign each of the seven agents to one of two abstract tiers — SimpleLLM for Kraljic/Spot Bidding/Award & Comms; DefaultLLM for Leverage, Bottleneck, Strategic, and Bid Evaluation agents — resolved to concrete Bedrock model IDs from `{env}-system-config` at instantiation via `DynamicAgentFactory`. The tier assignment follows cognitive demand: cheapest model capable of meeting the agent's quality bar as validated by the eval suite (PRD-008).

## Alternatives Considered

- **Single model for all agents.** Rejected: wastes per-token budget on high-volume templated agents that do not need top-tier reasoning; the cost differential is ~3–23× per token, making the saving too large to ignore.
- **Per-negotiation dynamic model selection.** Rejected: requires real-time quality prediction per agent invocation; adds latency and complexity, and the cognitive-demand classification is stable enough to set at design time (eval-validated, not assumed).
- **Status quo / no action.** Rejected: at one model for all agents the $0.10/negotiation target is not achievable without other mitigations; tiering is the primary structural lever.

## Trade-offs

| Gained | Given up |
| --- | --- |
| ~3× per-token saving (Sonnet→Haiku, Scenario A) to ~13–23× (Nova Pro→Lite/Micro, Scenario B) on high-volume agents, the single largest structural cost lever | Quality headroom on SimpleLLM agents — a mis-tiered agent cannot call a stronger model without a config change |
| Tier→model mapping is config-driven (`{env}-system-config`), so a mis-tier is correctable without redeploy | The tiering assumes the cognitive-demand classification is correct; the assumption is validated by the eval suite (AD-90, PRD-008), not hardcoded |

## Results

Per-agent tier table in PRD-009 §2 (SimpleLLM: Kraljic Classifier, Spot Bidding, Award & Comms; DefaultLLM: Leverage Auction, Bottleneck Negotiation, Strategic Partnership, Bid Evaluation). Tier labels are resolved by `DynamicAgentFactory` (AD-25, AD-65) at instantiation. The worked example in PRD-009 §2.2 reaches ~$0.095 on-demand and ~$0.061 after prompt caching (AD-59). The two concrete model families available — Scenario A (Claude Sonnet 4.5 / Haiku 4.5, ~3× gap, 1h cache TTL) and Scenario B (Nova Pro / Lite or Micro, ~13–23× gap, 5min TTL) — are the subject of the follow-on decision AD-58.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
