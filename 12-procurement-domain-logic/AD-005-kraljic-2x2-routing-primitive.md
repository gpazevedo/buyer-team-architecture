# AD-005 — Kraljic 2×2 Matrix Is the Core Routing Primitive

**Theme:** Procurement Domain Logic  **Catalog:** AD-5 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-11, AD-47, AD-60, AD-125, AD-152

## Context

Every purchase request must be routed to an appropriate negotiation strategy in a way that is deterministic, explainable to procurement professionals, and auditable. Without a routing primitive the workflow cannot branch to the correct Node-4 strategy agent, and routing decisions would be opaque or non-reproducible. Procurement theory already provides an established segmentation framework — the Kraljic matrix — which maps directly onto discrete strategies, making it a natural fit for a deterministic four-way branch.

## Decision

Use the Kraljic 2×2 (`profit_impact` × `supply_risk`) to classify each item into one of four quadrants (NON_CRITICAL, LEVERAGE, BOTTLENECK, STRATEGIC), each mapped to exactly one negotiation strategy (SPOT_BID, COMPETITIVE_AUCTION, PARTNERSHIP_RISK, PARTNERSHIP_VALUE). Thresholds default to 0.5 on each axis and are stored in `governance-policies.Kraljic_thresholds` in `{env}-system-config`, taking effect on next agent instantiation. This classification is the system's core routing primitive, driving the single deterministic four-way branch at Node 3.

## Alternatives Considered

- **ML-driven or free-form strategy selection.** Rejected: produces less explainable routing decisions that are harder to audit and reproduce.
- **Status quo / no explicit routing primitive.** Rejected: without a deterministic classification the workflow cannot branch to Node-4 strategy agents in a governed, repeatable way.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Recognized procurement framework maps cleanly onto a deterministic four-way branch | Nuance — a 2×2 is a deliberate simplification; items near a threshold boundary can flip quadrant |
| Routing decisions are explainable in domain terms to procurement professionals | Only four strategies exist; real-world cases outside the matrix must be handled by configuration tuning or escalation |
| Thresholds are tunable per tenant via configuration without code changes | Classifier unavailability requires a rule-based fallback (AD-47) that achieves 75–90% accuracy vs ~95% for the LLM classifier |

## Results

Realized as the four-way conditional branch at Node 3 (Strategy Router), producing exactly one Node-4 variant per negotiation (AD-11). Quadrant thresholds live in `governance-policies.Kraljic_thresholds` in `{env}-system-config`. A rule-based fallback covers classifier unavailability with quadrant-specific escalation for the riskier cases (AD-47). Kraljic classification results are semantically cached in `{env}-agent-session-cache` (24h TTL, keyed by `category_id` and threshold hash) to avoid re-classifying repeated categories (AD-60).

**Update 2026-08-22 — the 2×2 itself moves out of the prompt and into code; the model now only estimates the two axes (impl PR #354).** A `name_only` accuracy gate run came in at 14/20 = 70.0% against the 0.80 floor, and two of the six misses contradicted their own `reasoning` field — `Maintenance` ("high profit impact due to significant annual spend") and `Training` ("a non-critical area for profitability") were both submitted as BOTTLENECK, which is the **low** profit-impact quadrant, showing up in the confusion matrix as `STRATEGIC → BOTT:2`. The axis judgment was right; the table lookup was wrong. So `submit_kraljic_classification` no longer takes a `quadrant` argument at all: it takes `profit_impact` and `supply_risk`, and `response_builder` derives the quadrant through a new pure `tools.classify_quadrant`. This closes a quieter defect in the same move — the request's `classification_thresholds` previously reached the model only as prompt *text*, so a tenant override of `governance/default.Kraljic_thresholds` (this ADR's own threshold mechanism, AD-64) was honoured at the model's discretion; `classify_quadrant` now applies it deterministically, and an unset override still defaults to 0.5/0.5.

**The profit axis also could not resolve its own boundary.** `compute_history` conveyed profit impact as a 3-bucket `spend_tier` plus a `spend_trend` that cut at spend_tier's own 0.35/0.60 cutoffs — carrying no information the tier did not already have, with the middle bucket `[0.35, 0.60)` straddling the 0.5 quadrant boundary, leaving **36% of the calibration set undecidable on that axis**. The supply axis had never had this problem because `supply_chain_disruption_risk` was already on the same normalized scale as the threshold, which is exactly why supply-axis calls were the reliable half. `spend_trend` is replaced by `spend_materiality_index`, that signal's profit-axis counterpart on the same scale (`max |smi − p| = 0.103` across a 200-row fixture). Measured on a 200-row fixture: 81.7% → 85.5% Bayes-optimal, 80.7% → 85.1% under the plug-in 0.5 rule. Live, the `name_only` gate went 70.0% → 90.0% — but only after impl PR #356 fixed the harness that had been re-measuring the pre-deploy image ([AD-152](../09-infrastructure-deployment-platform-stack/AD-152-fresh-session-id-per-measurement-run.md)); the intermediate 80.0% reading was of the old image. **The standing rule for this primitive: the LLM estimates the axes, code applies the matrix** — see [AD-125](../02-agent-architecture-behavioral-control/AD-125-response-builder-replaces-structured-output.md) for the transcript read-back this rides on, and [AD-047](../06-reliability-resilience-graceful-degradation/AD-047-rule-based-kraljic-fallback.md) for the rule-based fallback, which was already doing exactly this.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
