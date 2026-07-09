# AD-092 — Cross-Family LLM-as-Judge Tier (`EvalLLM`)

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-92 · **Source PRD:** PRD-009, PRD-004 · **Status:** Accepted · **Related:** AD-57, AD-58, AD-59

> **Correction (2026-07-08):** the mechanism below — a `{env}-system-config`-resolved `EvalLLM` tier that inverts between Nova Pro and Claude Opus 4.5 depending on which "scenario" the agents run — was never built; there was never a Scenario A/B agent deployment to invert against (see AD-58's correction note). There *is* an `EvalLLM` tier seeded in `packages/buyer_agent_core/buyer_agent_core/seed_mirror.py` (`"EvalLLM": "us.meta.llama3-1-8b-instruct-v1:0"`), but grepping the codebase shows no agent or evaluator ever requests that tier — it's dead seed data, referenced only by its own unit test. The cross-family independence goal this ADR describes **is** actually achieved, just by a different, simpler mechanism: `evals/tone_negotiation_judge.py` hardcodes its own `EVAL_LLM_MODEL_ID` env var, defaulting to `us.anthropic.claude-sonnet-4-5-20250929-v1:0` (Claude Sonnet 4.5) — independent of `model_resolver.py`/`DynamicAgentFactory` entirely. Since all 6 live agents run on Nova (AD-57), a Claude-family judge is genuinely cross-family; the independence property holds, but through a standalone env var in the eval harness, not the tiering system this ADR names. The body below is preserved as the original (unbuilt) decision record.

## Context

LLM-as-Judge evaluators (Negotiation Quality, Communication Tone, Award Rationale Defensibility, and per-agent judge criteria in PRD-003 §2) score agent outputs as part of the evaluation pipeline (PRD-004 §4). Previously these evaluators ran on the same `DefaultLLM` tier as the agents they evaluated — under the deployed Scenario A, that meant a Claude Sonnet 4.5 model scoring Claude Sonnet 4.5 agent output.

This introduces a known bias: models exhibit self-preference and self-recognition effects when judging their own family's completions. The evaluator scores are inflated or skewed relative to what an independent judge would produce, degrading the signal quality of the online evaluation pipeline.

## Decision

Introduce a dedicated `EvalLLM` tier for LLM-as-Judge evaluators only. `EvalLLM` is always set to a *different model family* from the agent tiers:

- **Scenario A (deployed)** — agents on the Claude 4.5 family → `EvalLLM` = Amazon Nova Pro ($0.80 / $3.20 per 1M in/out). The cross-family judge is also cheaper per token.
- **Scenario B** — agents on Amazon Nova → `EvalLLM` = Claude Opus 4.5 ($5 / $25 per 1M in/out). The independence property is preserved, though at higher per-eval cost.

`EvalLLM` is resolved from `{env}-system-config` `model` item at evaluator instantiation (PRD-010 §2.2), the same resolution mechanism as the agent tiers. It tracks the agent scenario **inversely** — always the family the agents are not using — so the cross-family independence holds under both scenarios.

## Alternatives Considered

- **Single `EvalLLM` model (Nova Pro) for both scenarios.** Rejected: under Scenario B (Nova agents), a Nova Pro judge would be same-family with the agents it evaluates, losing the independence property. The inverse mapping avoids this without requiring a per-family specialization in judge infrastructure.
- **Run evaluators on `DefaultLLM` as before.** Rejected: same-family judging inflates scores by ~5–15% in known measurement (self-recognition bias), degrading the evaluation pipeline as a signal for automated actions (PRD-004 §4.5).
- **Use a dedicated open-weight model as judge.** Rejected: adds a third model provider and deployment surface; the cross-family Bedrock models already achieve independence without additional infrastructure.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Judge independence — scores are not inflated by self-recognition bias | Under Scenario B the judge (Opus 4.5) is more expensive per token than the agents |
| Under Scenario A the judge (Nova Pro) is cheaper per token than the agents it evaluates | `EvalLLM` adds one model-family to manage and monitor in the evaluation pipeline |
| Closed-loop quality automation (PRD-004 §4.5) receives unbiased signal | |

Under Scenario A (the deployed mapping), independence and cost savings align. Under Scenario B, independence is bought at a higher per-eval cost — an accepted trade-off prioritising judge independence over eval cost.

## Results

`EvalLLM` defined in PRD-009 §2.0, registered in PRD-004 §4.3, and resolved from `{env}-system-config` `model` item per PRD-010 §2.2. The cross-PRD invariants table records the `EvalLLM` row with its Scenario A/B inverse mapping. CI/E2E `SimpleLLM` references in PRD-008 are unaffected (those are ground-truth golden-dataset checks, not LLM-as-Judge).

**What's actually true:** the real judge model default lives in `evals/tone_negotiation_judge.py:20` (`EVAL_LLM_MODEL_ID`, default Claude Sonnet 4.5), consumed by `score_negotiation_quality()`/`score_communication_tone()` (PRD-004 §4.3.5). It is not registered in any system-config `model` item and has no Scenario A/B inversion — there being only one agent family (Nova) to be independent from. The unused `seed_mirror.py` `EvalLLM`=Llama entry and this real Claude-default env var are two different things that happen to share a name.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
