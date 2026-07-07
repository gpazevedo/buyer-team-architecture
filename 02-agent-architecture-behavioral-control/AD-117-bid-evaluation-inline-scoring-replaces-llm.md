# AD-117 — Bid-Evaluation AgentCore Runtime Removed; Inline Deterministic Scoring Replaces the LLM Agent

**Theme:** Agent Architecture & Behavioral Control
**Catalog:** AD-117 · **Source PRD:** PRD-002, PRD-003 · **Status:** Accepted · **Related:** AD-021, AD-023, AD-096

## Context

The Bid Evaluation agent (`agents/bid_evaluation_llm/`) was the 5th LLM agent in the 7-node DAG — it consumed the agent-scored bids from Node 4x and produced composite evaluation scores (bid format compliance, governance policy compliance, negotiation quality composite). Each invocation took 25–60s of LLM time plus microVM startup, and the evaluation task itself is inherently deterministic: check that bid fields are present and well-formed per the response schema, check that governance policies (TCO completeness, risk assessment, relationship history) were satisfied in the agent's tool call record, compute a weighted composite. An LLM adds latency and cost with no accuracy gain — there is no subjective judgement or natural-language reasoning in evaluating whether a Pydantic-schema field is present or a steering guard fired.

AD-096 already established the precedent for inline deterministic logic in a previously-LLM step (Node 2 Kraljik short-circuit from seeded `Category.kraljic_quadrant`). The bid evaluation step is an even stronger case: the evaluator checks deterministic structural properties of its upstream agents' own structured outputs, no classification judgement is involved.

## Decision

Remove the `bid_evaluation_llm` AgentCore runtime entirely (code, Terraform, CI, ECR repo, IAM role, CloudWatch alarms). Replace with `orchestrator/scoring.py` — a synchronous, inline `score_bid_evaluation()` function combining bid format compliance checks and governance policy compliance checks into a single deterministic pass within the Node 5 Lambda handler. The composite score computation (weighted average of constituent scores) is inlined rather than delegated to an LLM.

Also bundled in the same change (PR #151): low-value SPOT_BID and COMPETITIVE_AUCTION items under `SPOT_BID_MAX_VALUE` ($5k) now skip the LLM agent entirely in `node_strategy_execute.py`, pricing deterministically (~5s vs ~45s). Spot-bidding tier downgraded from StrategyLLM to DefaultLLM (Nova Lite) for the remaining above-threshold path — its single-supplier direct-pricing task doesn't need Nova Pro.

## Alternatives Considered

- **Keep the LLM agent but optimize with caching.** Rejected: bid evaluation runs once per negotiation on unique bid sets; semantic caching would have near-zero hit rate. The latency is structural — it's a synchronous step in the hot path blocking the approval gate.
- **Move evaluation to an async evaluator Lambda (like bid format compliance).** Rejected: the composite score gates the approval decision (Node 6 reads it before auto-approve/pause), so it cannot be async — it must complete before the SFN advances past Node 5.
- **Keep the agent but use a smaller model.** Rejected: the evaluation task doesn't benefit from any model size — it's a deterministic check, not a reasoning task.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Node 5 evaluation step drops from 25–60s to <1s — the negotiation hot path shortens by ~30s | Bid evaluation quality assessment is now purely structural — it checks what the response schema guarantees, not what an LLM could infer about negotiation quality from the response text |
| One fewer AgentCore runtime to deploy, monitor, alarm, and pay session time for (6 agents instead of 7) | The `negotiation_quality_composite` now derives purely from structural compliance scores, not LLM-judged negotiation quality — the LLM-as-Judge evaluator (AD-032, online eval on `EvalLLM`) is the separate quality signal |
| No regression risk from model updates changing evaluation behaviour | A future qualitative dimension (e.g., "did the agent negotiate creatively") would need a new evaluator, not just a prompt change |

## Results

Shipped in PR #151 (merged 2026-07-06). 164 tests pass, clean terraform plan. Dev resources destroyed: IAM role + 4 policies, ECR repo + lifecycle policy, 2 CloudWatch alarms, AgentCore runtime. The 6 remaining LLM agents are: Kraljic Classifier, Spot Bidding, Leverage Auction, Bottleneck Negotiation, Strategic Partnership, Award & Comms. The inline pattern established here applies to any future agent step whose task is deterministic structural validation of structured outputs.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
