# AD-062 — Batch Inference for Nightly Supplier KPI Recalculation

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-62 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-57, AD-59, AD-46

## Context

Supplier KPI recalculation is a large, periodic, read-only workload: it processes all supplier performance records to update scoring inputs used by negotiation agents. It has no latency requirement — KPIs used in a negotiation starting at 09:00 UTC can tolerate being based on data computed at 02:00 UTC. Running this workload through on-demand Bedrock inference pays the full synchronous invocation price for a workload that does not need synchronous latency. Bedrock Batch offers a 50% discount over on-demand rates, with the trade-off that batch jobs have their own scheduling and failure characteristics and results are not available immediately.

## Decision

Run supplier KPI recalculation via Bedrock Batch (50% discount vs on-demand), scheduled nightly at 02:00 UTC. Feature-flagged (`batch_supplier_evaluation_enabled`, default off) so the default deployment does not depend on batch infrastructure and individual environments can opt in via system-config.

## Alternatives Considered

- **On-demand synchronous inference for KPI recalculation.** Rejected: pays full on-demand rates for a workload with no latency requirement; the 50% batch discount is available at no quality cost for this workload.
- **Skip nightly recalculation (use stale supplier KPIs).** Rejected: supplier KPIs are a material input to bid evaluation and award decisions; skipping recalculation entirely degrades negotiation quality beyond acceptable bounds.
- **Always-on batch (no feature flag).** Rejected: makes the default deployment dependent on batch infrastructure and Bedrock Batch availability; a feature flag allows incremental rollout and environment-specific opt-in consistent with the platform's feature-flag lifecycle (AD-66).

## Trade-offs

| Gained | Given up |
| --- | --- |
| 50% inference discount on a workload with no latency requirement — the only cost lever that trades freshness for price | Results are up to ~24 hours stale relative to real-time on-demand inference; batch job failures require monitoring and retry outside the normal request path |
| Default-off feature flag means the default deployment does not depend on batch infrastructure; environments opt in explicitly | The default deployment foregoes the 50% saving until the flag is enabled; this is an intentional safety choice, not an oversight |

## Results

Metric `batch.invocations_saved` (dimensioned by `tenant_id`) tracks realized savings. Enabled in prod via system-config feature flag `batch_supplier_evaluation_enabled`. This is the fourth and only lever in the cost-optimization stack (after model tiering AD-57, prompt caching AD-59, and semantic caching AD-60) that trades a quality property — freshness — for a price reduction. The graceful-degradation posture for non-real-time quality signals is consistent with AD-46 (memory failures never block); nightly KPI results are a quality enhancement, not a hard dependency for negotiation correctness.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
