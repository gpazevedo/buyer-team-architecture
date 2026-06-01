# AD-009 — PR→PO Splitting: One PO per Supplier with ≥1 Awarded Item

**Theme:** Procurement Domain Logic  **Catalog:** AD-9 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-10, AD-11, AD-19

## Context

A single Purchase Requisition can award different line items to different suppliers. Downstream ERP and P2P systems model Purchase Orders scoped to one supplier — a PR-level granularity does not map cleanly to those systems. Without an explicit splitting rule the Node 7 PO assembly step has no deterministic way to produce correctly scoped PO entities, and the authorization snapshot to attach to each PO is ambiguous.

## Decision

After all awards are finalized, group awarded items by `supplier_id`; each supplier holding ≥1 awarded item receives exactly one PO. Each PO inherits a snapshot of the parent PR's `read_claims` and `approve_claims` (copied, not referenced) so authorization is deterministic for the PO's lifetime regardless of later PR changes, and inherits the PR's currency under the one-PR-one-currency rule. PO delivery address is also inherited from the PR. PO status is set to ISSUED on creation.

## Alternatives Considered

- **One PO per PR.** Rejected: cannot represent a multi-supplier award where different items go to different suppliers.
- **One PO per awarded line item.** Rejected: produces unnecessary PO proliferation and does not match standard ERP modeling of supplier-level purchase orders.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Clean, deterministic award-to-order mapping that matches how ERPs model POs | One-to-one PR↔PO simplicity — a single PR can fan out into many POs |
| Authorization claims are snapshotted at PO creation, making each PO self-contained | Increased number of PO entities and bookkeeping surface around partial awards |
| Fulfillment, acknowledgement, and export logic remain scoped to one supplier per PO | — |

## Results

Realized at Node 7 (Award and Comms / PO Assembly), where the splitting rule groups all `Award` records by `supplier_id` and emits one `PurchaseOrder` entity per group into `{env}-orders`. Each PO carries the inherited currency, delivery address, and authorization claim snapshot. One PR can produce multiple POs (fan-out); the `{env}-orders` table's `supplier_index` GSI supports supplier-scoped queries over the resulting PO set. The Node-7 agent is the only place PO entities are created, keeping the splitting rule a single governed code path.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
