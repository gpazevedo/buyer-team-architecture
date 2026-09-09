# AD-159 — SAP Catalog Price Grounds the Bid-Evaluation Baseline; the Budget Cap Stays on the Requisitioner's Ask

**Theme:** Procurement Domain Logic
**Catalog:** AD-159 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-118, AD-155, AD-158

## Context

`node_bid_evaluation._get_market_benchmark` averages each negotiation item's `estimated_unit_price` into `market_avg_price`, one of `scoring.score_bid_dimensions`'s inputs for ranking submitted bids — a benchmark a supplier's bid is scored against, not a cap on what an agent may offer (that's `budget_limit`, AD-118's walk-away ceiling, a separate field entirely). Before this decision, `sap_ingest.map_sap_requisition` set every item's `estimated_unit_price` to the SAP PR line's own `PurchaseRequisitionPrice` — whatever price the requisitioner typed when raising the requisition. That is a self-reported, requisitioner-supplied number, not an independent reference: scoring a bid against the price the requisitioner already expected to pay is close to scoring it against nothing. Gap-closure plan Phase 4.1's S/4-shaped surface migration (AD-158) exposed `A_Product` — SAP's own materials master, carrying each material's own `unitPrice` — as a first-class entity for the first time; nothing in the ingest path read it yet.

## Decision

Add `sap_ingest.fetch_material_prices`, an unfiltered `GET A_Product` returning `{Product: unitPrice}` for the mock's seed catalog — the same "the seed set is small enough to fetch in full" idiom `sap_supplier_ingest.fetch_sap_suppliers` already uses. `map_sap_requisition` takes an optional `material_prices` map and, per line, prefers the catalog's `unitPrice` for that line's `Material` over `PurchaseRequisitionPrice` when the material has a catalog entry; a line whose material isn't in the map, or a caller that passes none at all, keeps today's `PurchaseRequisitionPrice` behavior unchanged. Both ingest entry points wire this the same way, fail-open: `sap_pr_ingest_handler._fetch_material_prices` (direct-HTTP orchestrator path) and `skill_runtime/server.py`'s `ingest_purchase_requisitions_sap` (Gateway-routed path) each wrap the fetch so any exception is logged and swallowed, and `map_sap_requisition` falls back to `PurchaseRequisitionPrice` on the resulting empty map rather than aborting ingest.

`budget_override` — the requisition-level total `map_sap_requisition` also computes — deliberately keeps summing `PurchaseRequisitionPrice` regardless of `material_prices`. The two fields now answer two different questions from two different sources on purpose: `estimated_unit_price` (catalog-grounded when available) asks "is this bid competitive against an independent reference"; `budget_override`/`budget_limit` (always the requisitioner's own figure) asks "did the requisitioner authorize spending this much." Feeding the catalog price into the budget side too would let SAP's list price silently override what a human actually approved to spend.

## Alternatives Considered

- **Feed `A_Product.unitPrice` into `budget_override`/`budget_limit` as well, on the reasoning that SAP's own price is more authoritative everywhere.** Rejected: `budget_limit` is a human authorization ceiling, not a market signal (AD-118) — substituting SAP's catalog list price would let an unattended master-data value silently raise or lower what an agent is allowed to spend, without the requisitioner ever approving the new number.
- **Always prefer the catalog price and drop the `PurchaseRequisitionPrice` fallback entirely.** Rejected: the mock's seed catalog — and any early real S/4 tenant — won't have every requisitioned material cataloged. A line with no reference price would need `estimated_unit_price` to go unset, which `_get_market_benchmark` already treats as "no signal" — worse than the number the PR line already carries.
- **Fetch material prices per line (one OData read per requisitioned material) instead of the full catalog once per ingest.** Rejected: same reasoning `fetch_sap_suppliers` already established for the supplier pool — the mock's seed set is small enough that one unfiltered read is cheaper and simpler than N filtered ones, and a real S/4 tenant's materials master is exactly the kind of reference data worth caching in full rather than probing per line.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Bid scoring now has an independent reference price (SAP's own catalog) instead of scoring bids against the number the requisitioner already expected | A silent fallback: an ingest where `fetch_material_prices` fails logs a warning and proceeds on `PurchaseRequisitionPrice` alone, so a degraded catalog read is not visibly distinguishable from "this material genuinely isn't cataloged" without reading the log |
| `budget_override`'s authorization semantics stay exactly what they were — no behavior change on the spend-ceiling side of the pipeline | Two different "price" concepts now live on the same ingested requisition (`estimated_unit_price` vs. the total behind `budget_override`), which a future reader could conflate without this ADR's context |
| Reuses an established idiom (`fetch_sap_suppliers`'s full-catalog read) rather than inventing a second caching strategy | No dedicated test yet asserts the catalog price actually wins over `PurchaseRequisitionPrice` on a matched material — see Results |

## Results

`orchestrator/sap_ingest.py` (`fetch_material_prices`; `map_sap_requisition`'s `material_prices` parameter and per-line `baseline_price` selection), `sap_pr_ingest_handler.py` (`_fetch_material_prices`, fail-open wrapper), `skill_runtime/server.py` (`ingest_purchase_requisitions_sap`, same fail-open wrapper). Shipped impl PR #450 (gap-closure plan 5.2); tests landed alongside it in `orchestrator/tests/test_sap_ingest.py`, `test_sap_pr_ingest_handler.py`, and `skill_runtime/tests/test_ingest_sap.py`, exercising the new optional parameter's default (`None` → today's behavior unchanged) and the fetch-failure fallback path — not yet a direct assertion that a matched material's catalog price overrides its `PurchaseRequisitionPrice` in the output row. One day later, impl PR #452 (AD-158's Phase B) gave `fetch_material_prices` an MCP-transport counterpart in `sap_odata_mcp.py`, so both ingest entry points now resolve this same baseline through either transport, not direct-HTTP only. Downstream, `node_bid_evaluation._get_market_benchmark` reads whatever `estimated_unit_price` ends up on the `{env}-items` row without knowing which source produced it — this decision changes an upstream input to an existing scoring path, not the scoring path itself.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
