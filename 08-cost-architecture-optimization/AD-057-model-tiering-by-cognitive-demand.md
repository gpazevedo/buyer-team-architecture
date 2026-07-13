# AD-057 — Model Tiering by Cognitive Demand

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-57 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-21, AD-58, AD-25, AD-65

## Context

Without a decision, every agent uses the strongest (and most expensive) model available. High-volume, templated work — Kraljic classification, spot bids, award notifications — does not require the reasoning power of the most capable model. A worst-case negotiation without any optimization costs $1.50–$3.00 (PRD-001 §2.2 KR3.2 sets a $0.10 average target). Model selection is the single largest structural lever available before caching, semantic caching, or batch inference are applied, because it reduces the per-token base rate on the agents that dominate token volume.

## Decision

Assign each agent to an abstract tier — resolved to concrete Bedrock model IDs from `{env}-system-config` at instantiation via `DynamicAgentFactory`. The tier assignment follows cognitive demand: cheapest model capable of meeting the agent's quality bar as validated by the eval suite (PRD-008).

The original two-tier split (SimpleLLM for Kraljic/Spot Bidding/Award & Comms; DefaultLLM for Leverage, Bottleneck, Strategic, and Bid Evaluation) proved too coarse under live e2e: **Spot Bidding** and **Bid Evaluation** were promoted to a third `StrategyLLM` tier (Nova Pro) — the tier the strategic negotiation agents already use — because their weaker tiers failed the reliability bar (see Results). The tier→model mapping being config-driven is exactly what let these corrections ship without a code change.

> **Correction (2026-07-08):** this decision and its "Results" section describe the tier mapping as of PR #87/#88. It has since moved three times more — see the notes appended to Results below. There was never a second, Claude-based model family in production; the "Scenario A/B" framing this ADR and AD-58 used to describe that choice has been retired — see AD-58's correction note.

## Alternatives Considered

- **Single model for all agents.** Rejected: wastes per-token budget on high-volume templated agents that do not need top-tier reasoning; the cost differential is ~3–23× per token, making the saving too large to ignore.
- **Per-negotiation dynamic model selection.** Rejected: requires real-time quality prediction per agent invocation; adds latency and complexity, and the cognitive-demand classification is stable enough to set at design time (eval-validated, not assumed).
- **Status quo / no action.** Rejected: at one model for all agents the $0.10/negotiation target is not achievable without other mitigations; tiering is the primary structural lever.

## Trade-offs

| Gained | Given up |
| --- | --- |
| ~13–23× per-token saving (Nova Pro→Lite/Micro) on high-volume agents, the single largest structural cost lever | Quality headroom on SimpleLLM agents — a mis-tiered agent cannot call a stronger model without a config change |
| Tier→model mapping is config-driven (`{env}-system-config`), so a mis-tier is correctable without redeploy | The tiering assumes the cognitive-demand classification is correct; the assumption is validated by the eval suite (AD-90, PRD-008), not hardcoded |

## Results

Per-agent tier table in PRD-009 §2 (originally SimpleLLM: Kraljic Classifier, Spot Bidding, Award & Comms; DefaultLLM: Leverage Auction, Bottleneck Negotiation, Strategic Partnership, Bid Evaluation — see the two revisions below for what actually shipped). Tier labels are resolved by `DynamicAgentFactory` (AD-25, AD-65) at instantiation. The worked example in PRD-009 §2.2 reaches ~$0.095 on-demand and ~$0.061 after prompt caching (AD-59). Only one model family was ever deployed — Amazon Nova (Pro/Lite/Micro), ~13–23× gap, 5-minute cache TTL. AD-58 originally proposed a second, Claude-based family ("Scenario A"); that was never built (see AD-58's correction note).

**As-built revision 1 (PR #87, #88).** Full live e2e exposed two mis-tiers, both fixed by promoting the agent to `StrategyLLM` (Nova Pro): **Bid Evaluation** on Nova Lite intermittently dropped required structured-output fields (`ranked_bids[].supplier_id`), failing the agent run → Node 5 fallback → no quality composite; **Spot Bidding** on Nova Micro intermittently collected zero bids in its tool loop, so the negotiation ended `REQUIRES_ATTENTION` (`entry_trigger=zero_bids`) with no PO. Both are the "mis-tier corrected via config, assumption validated by the eval/e2e suite" trade-off in this decision playing out — the cognitive-demand classification was too optimistic for these two agents on the weaker Nova models, and the correction was a `{env}-system-config` tier change, no redeploy of the tiering logic.

**As-built revision 2 (PR #151, merged 2026-07-06; AD-117).** Both agents this ADR's tier-promotion once applied to have since changed again, superseding revision 1 above: the `bid_evaluation_llm` runtime was deleted outright and replaced with deterministic inline scoring (`orchestrator/scoring.py`) — there is no tier assignment for Bid Evaluation at all now, because there is no LLM call to tier. **Spot Bidding** was downgraded from `StrategyLLM` back down to `DefaultLLM` (Nova Lite, confirmed in `agents/spot_bidding_llm/agent_spec.py:64`) for its remaining above-`SPOT_BID_MAX_VALUE` ($5k) path; below that threshold it now also skips the LLM entirely and prices deterministically.

**As-built revision 3 (PR #211, merged 2026-07-13).** Live smoke-testing surfaced a third mis-tier, same failure shape as revision 1 (weak-tier model reliability, not a code defect): **Kraljic Classifier** on `SimpleLLM` (Nova Micro) scored 80% then 60% against `smoke_kraljic`'s 100%-with-axes bar over two live runs — confirmed model judgment, not a field-mapping bug, because `response_builder` (AD-125) read back exactly what the model argued to `submit_kraljic_classification` on every call, and different categories failed on different runs. Promoted to `DefaultLLM` (Nova Lite): 3 live runs at `-n 10` scored 100%, 100%, 90% (29/30 ≈ 97%) — error rate dropped from ~30% to ~3%. The 6 live LLM agents and their current tiers are **Kraljic Classifier (`DefaultLLM`)**, Spot Bidding (`DefaultLLM`), Leverage Auction / Bottleneck Negotiation / Strategic Partnership (`StrategyLLM`), Award & Comms (`SimpleLLM`, configured but its runtime is not invoked on the live path — see PRD-009 §2 note).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
