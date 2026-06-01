# AD-078 — Synthetic GSI Sort Keys and Composite Cursor Pagination

**Theme:** Test Tenant Platform & Data
**Catalog:** AD-78 · **Source PRD:** PRD-015 · **Status:** Accepted · **Related:** AD-77

## Context

The `tenant-mdm-emulator` MCP server (AD-77) must return entities in a stable, paginated order to faithfully emulate an ERP read API. DynamoDB does not guarantee ordering within the same partition key beyond the sort key, and multiple writes within the same millisecond have no inherent order. Naive `LastEvaluatedKey`-based pagination can skip or repeat rows when the underlying data changes between pages.

## Decision

Add synthetic GSI sort keys — `lm_sk` (`"{last_modified}#{entity_id}"`), `cat_sk` (`"{category_id}#{item_id}"`), and `status_sk` (`"{status}#{last_modified}"`) — populated atomically with their underlying fields on every `PutItem` and `UpdateItem`. Use composite cursor pagination keyed on `(last_modified, entity_id)`, base64-encoded as `page_token`, with the query predicate `tenant_id=X AND (last_modified, entity_id) > cursor`.

## Alternatives Considered

- **Native DynamoDB `LastEvaluatedKey` pagination.** Rejected: unstable under concurrent modification — pages can skip or duplicate rows when items are written between page fetches.
- **Offset-based pagination (skip/limit).** Rejected: DynamoDB has no native offset; implementing it requires full scans or separate count tracking, both expensive and impractical at ~541k rows.
- **Sort by a simple timestamp attribute without a tiebreaker.** Rejected: same-millisecond writes are not uniquely ordered, producing non-deterministic results that violate ERP read semantics.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Deterministic ordering even for same-millisecond writes — the `entity_id` tiebreaker in the composite cursor breaks all ties | Extra write cost: every entity write must populate synthetic sort-key attributes on every mutation or GSI rows become stale |
| Stable cursor pagination that does not skip or duplicate rows under concurrent modification | Increased schema surface: three synthetic attributes (`lm_sk`, `cat_sk`, `status_sk`) exist solely to support ordered reads |
| Faithfully reproduces the stable pagination a real ERP read API provides | Composite-cursor scheme is more complex to implement and test than offset pagination |

The synthetic sort-key attributes are not optional — every writer using `dynamodb-master-data` must set them atomically on every mutation. This invariant is enforced at the MCP server level, not by convention.

## Results

The synthetic sort keys back the `list_categories`, `list_suppliers`, `list_items`, and `list_purchase_requisitions` tools in the `tenant-mdm-emulator` MCP server (AD-77). The composite `(last_modified, entity_id)` cursor is the pagination contract for all ordered reads consumed by the Test Tenant App's dataset visualization views (PRD-013) and by Buyer Team's `ingest_*` tools when syncing master data incrementally via the `since` parameter. Detail is owned by PRD-015 §3.1 and §3.3.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
