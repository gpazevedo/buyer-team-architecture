# AD-096 — Node 2 Kraljic Live-Schema Hybrid Short-Circuit

**Theme:** Procurement Domain Logic  **Catalog:** AD-96 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-5, AD-47, AD-60, AD-11

## Context

Node 2 of the 7-node DAG is responsible for Kraljic classification: it determines which procurement quadrant (NON_CRITICAL, LEVERAGE, BOTTLENECK, STRATEGIC) a category belongs to, driving the 4-way branch at Node 3. The original design ran every category through the semantic cache, then the Kraljic Classifier Agent on a cache miss, then the rule-based fallback (AD-47) on agent unavailability. This assumes incoming categories carry raw `profit_impact` and `supply_risk` axes that must be classified at runtime.

The live `{env}-categories` schema, however, pre-classifies items: test-tenant categories carry a seeded `kraljic_quadrant` field. Running the cache+agent path for categories already bearing a quadrant wastes an LLM call (and semantic cache write) on a value that was already determined at data-seeding time. The live schema also surfaces two secondary gaps: categories without axis values (missing `profit_impact` / `supply_risk`) that the cache-key hash would fail on, and confirmed that the cache store is the generic `{env}-agent-session-cache` table (`pk`/`sk` + `ttl`), not a composite-named key.

This decision was shipped in PRD-002 v1.5.0 and validated on `dev` (2026-06-13). It was held as candidate C-8 in the decisions catalog pending elevation to architecture scope; it is promoted here because it defines a durable data contract (seeded quadrant = authoritative; no-quadrant = classify at runtime) that all future tenants and Node 2 implementations must observe.

## Decision

Node 2 implements a **hybrid short-circuit**: when the inbound `Category` carries a seeded `kraljic_quadrant` value, Node 2 uses it directly as the classification result (confidence from `classification_confidence_score` if numeric, else `1.0`) and skips the semantic cache check, the A2A agent call, and the rule-based fallback. The cache+agent+fallback path is exercised only for an unclassified category (no `kraljic_quadrant` field), which is the expected state for categories arriving from a real tenant's ERP integration.

Three supporting rules travel with this decision:

- **Missing axes:** If a category lacks `profit_impact` or `supply_risk`, both are read as `0.0` via `category.get(..., 0)`. This is deterministic: `(0.0, 0.0)` maps to NON_CRITICAL under the standard Kraljic thresholds, the safest classification (lowest disruption if wrong).
- **Cache store identity:** The semantic cache for Kraljic results (AD-60) uses the generic `{env}-agent-session-cache` table with `pk`/`sk` keys plus a `ttl` attribute — not a composite table name. No code should construct a Kraljic-specific table name.
- **Seed nothing:** Node 2 adapts to the live schema rather than seeding missing fields. The short-circuit exists because the data is already classified, not because Node 2 should seed classifications.

## Alternatives Considered

- **Always run the cache+agent path; ignore the seeded quadrant.** Rejected: wastes an LLM call on a pre-determined value; the seeded quadrant is authoritative for the test tenant and any future tenant that pre-classifies categories at ERP ingest time.
- **Remove the cache+agent path for all tenants in v1.0.** Rejected: the non-short-circuit path is the correct path for unclassified categories from a real ERP; it must remain fully operational for real-tenant use.
- **Seed missing axis values into the category record.** Rejected: violates "seed nothing" — it would corrupt the source-of-truth record with synthetic data; reading `0.0` at classification time is sufficient and leaves the record clean.

## Trade-offs

| Gained | Given up |
| --- | --- |
| O(1) Node 2 cost for pre-classified categories: no cache write, no A2A call, no LLM charge | Node 2 now branches on the presence of `kraljic_quadrant`; future schema changes to that field name are a breaking change to Node 2 |
| Deterministic outcome for any category: seeded quadrant → use it; no quadrant → classify it; missing axes → `0.0` = NON_CRITICAL | The `0.0 → NON_CRITICAL` mapping is only as correct as the threshold defaults; a config change to the `profit_axis` / `supply_axis` thresholds could change the no-axes classification without any category data change |
| The short-circuit is transparent to downstream nodes — the same `Negotiation.kraljic_quadrant` field is set regardless of which path ran | |

The hybrid design means the test tenant and production real-tenant paths exercise different Node 2 branches. Integration tests must cover both: the short-circuit path (seeded quadrant) and the full path (unclassified category, agent call, cache miss/hit).

## Results

Realized in PRD-002 §3.2 (Live-schema reconciliation note, impl 2026-06-13) and PRD-002-impl §3.2. The short-circuit is gate-free: no feature flag is needed because the logic is purely data-driven (`if category.kraljic_quadrant` is present, use it). AD-60 (semantic cache for Kraljic) and AD-47 (rule-based fallback) remain in effect for the unclassified path; they are not replaced by this decision. AD-5 (Kraljic 2×2 as the routing primitive) is unchanged: this decision governs only *how* the quadrant is resolved at runtime, not the quadrant-to-strategy mapping.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
