# AD-060 — Semantic Cache for Kraljic Results

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-60 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-5, AD-59, AD-57

## Context

The Kraljic Classifier agent maps a procurement category to one of four quadrants based on `(profit_impact, supply_risk, thresholds)`. For the same category and the same threshold configuration the result is deterministic — the LLM is not adding new reasoning, it is re-deriving the same quadrant. Repeated classification of the same category is pure waste that compounds at scale when many negotiations share the same category. With prompt caching already active (AD-59), each Kraljic LLM call that does hit the cache costs only ~$0.0002 in input tokens; a semantic-cache hit eliminates the LLM call entirely and saves ~$0.0008 per classification. Invalidation must be automatic when thresholds change, because a manual flush is an operational error surface.

## Decision

Cache Kraljic classification results in DynamoDB (`{env}-agent-session-cache`, PRD-007 §3.1) with a composite key: PK `{tenant_id}#kraljic_classifier`, SK `{category_id}#{hash(profit_impact, supply_risk, thresholds)}`. A per-item TTL of 24 hours (via DynamoDB TTL attribute) overrides the table's 72-hour default for these entries. A threshold change changes the hash, automatically invalidating the cache entry without a manual flush. Feature-flagged (`semantic_caching_enabled`, default true).

## Alternatives Considered

- **No semantic cache (rely on prompt caching alone).** Rejected: prompt caching saves on the token cost of a cache read but does not eliminate the LLM call itself; for high-category-reuse workloads the repeated invocations add up, and the elimination is essentially free to implement alongside an existing cache table.
- **In-memory agent-level cache.** Rejected: does not survive across invocations or agents; provides no durability and no tenant isolation, and is incompatible with the ephemeral-session model (AD-2).
- **Longer TTL (72-hour or indefinite).** Rejected: a category's quadrant assignment should reflect current threshold configuration; a 24-hour TTL bounds how stale the cached result can be while still providing meaningful hit rates for active categories.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Eliminates redundant Kraljic LLM calls on repeated categories; a hash-in-key design invalidates automatically on threshold or axis change — no manual flush required | A minor lever: with prompt caching already active each semantic-cache hit saves only ~$0.0008 — sub-cent — so this is not a primary cost driver |
| 24h TTL bounds staleness; the cache entry is consistent within a threshold-configuration epoch | Adds a cache table dependency and an invalidation surface to reason about; the 24h per-item TTL must be correctly applied to override the table's 72h default |

## Results

Physical key: PK `{tenant_id}#kraljic_classifier`, SK `{category_id}#{hash}`. Stored in `{env}-agent-session-cache` (PRD-007 §3.1). Feature flag `semantic_caching_enabled` (default true) in system-config. The 24h per-item TTL is applied via the DynamoDB TTL attribute and overrides the table's 72h default for Kraljic entries. With both prompt caching (AD-59) and a semantic-cache hit, the Kraljic step costs ~$0.0006 vs $0.0016 uncached, contributing the marginal ~$0.0008 saving that takes the worked example from ~$0.0614 to ~$0.0606 (both round to $0.061 at three-decimal precision). The four-quadrant routing that this cache result feeds is the subject of AD-5 and the DAG branch defined in AD-11.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
