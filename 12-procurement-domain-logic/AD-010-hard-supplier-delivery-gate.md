# AD-010 — Hard Supplier Delivery Gate Before Invitation

**Theme:** Procurement Domain Logic  **Catalog:** AD-10 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-5, AD-11, AD-18

## Context

Inviting a supplier that cannot physically deliver to the PR's address, or that cannot meet the required delivery window, wastes a full negotiation cycle — agent invocations, supplier communications, and elapsed calendar time — and pollutes the bid pool with bids that cannot be awarded. Without an explicit gate before invitation, undeliverable suppliers enter strategy execution and are only discovered during bid evaluation, at maximum cost.

## Decision

Apply a hard gate at the end of Node 3 (Strategy Router), after the quadrant-specific candidate pool is assembled, before any supplier is invited. A supplier must pass two checks: (1) it can serve the PR's `delivery_address` (via `can_deliver_to(address)`), and (2) it meets `delivery_threshold_days` (the hard cutoff). Failing suppliers are discarded and recorded in `Negotiation.discarded_suppliers` with a typed reason (ADDRESS or THRESHOLD). If the gate leaves the pool empty, the negotiation transitions to CANCELLED with `cancellation_reason = "no_eligible_suppliers"` and the PR is also set to CANCELLED.

## Alternatives Considered

- **Soft delivery scoring only.** Rejected: would admit suppliers that physically cannot deliver into the negotiation, wasting cycles on bids that can never be awarded.
- **Gate at bid evaluation (Node 5) rather than before invitation (Node 3).** Rejected: defers the discard to maximum cost — agent invocations and supplier communications have already been spent.

## Trade-offs

| Gained | Given up |
| --- | --- |
| No negotiation effort or cost is spent on suppliers that cannot deliver | Competitive breadth — a supplier marginally over the threshold is excluded outright even if otherwise attractive |
| The discard set is recorded and auditable (`Negotiation.discarded_suppliers`) | — |
| The empty-pool failure case is explicit (CANCELLED with typed reason) rather than a silent stall | — |

The design mitigates the competitive-breadth loss by separating the hard `delivery_threshold_days` (a binary gate) from the soft `delivery_ideal_days` (a scoring input at Node 5 bid evaluation): "close but late" improves a bid's score rather than blocking it; only suppliers that cannot meet the cutoff at all are excluded.

## Results

The gate runs at the end of Node 3, after Kraljic classification (AD-5) and candidate-pool assembly (AD-11), and works in parallel with the two-tier ESG enforcement (candidate-pool gate at Node 3 plus bid-evaluation weighting, AD-18). The `{env}-negotiations` table records discarded suppliers; the typing of the reason (ADDRESS vs THRESHOLD) gives ops per-condition visibility. The CANCELLED-on-empty-pool path is excluded from the Award Rate KPI denominator, keeping that metric meaningful.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
