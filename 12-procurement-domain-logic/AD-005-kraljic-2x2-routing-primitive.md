# AD-005 — Kraljic 2×2 Matrix Is the Core Routing Primitive

**Theme:** Procurement Domain Logic  **Catalog:** AD-5 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-11, AD-47, AD-60

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

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
