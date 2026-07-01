# AD-109 — PO Receiving Lifecycle: RECEIVED Terminal State + Typed Ack/Reject + Trace Chain

**Theme:** Procurement Domain Logic
**Catalog:** AD-109 · **Source PRD:** PRD-013 · **Status:** Accepted · **Related:** AD-97, AD-9, AD-76, AD-17

## Context

The orchestrator's job ends when Node 7 issues a Purchase Order (`ISSUED`, one per supplier — AD-9) and the durable outbox hands it off for delivery (AD-97). But a delivered PO is not the end of its life: the buyer's receiving domain (PRD-013) still has to model whether the PO was **acknowledged** or **rejected**, and by whom, with what reason — and a buyer looking at a PO must be able to trace it back to the requisition, negotiations, and award that produced it (the identity chain of AD-76). Before this decision the orchestrator treated `ISSUED` as terminal and tests asserted it, so a delivered PO had no downstream state, acknowledgment/rejection had nowhere to live, and the origin trace was unreachable from the order side. A first attempt (PR #92) collected a rejection reason in the UI but dropped it on the way back — write-only dead data — because no layer between DynamoDB and the SPA declared the fields.

## Decision

The delivered PO has its own **receiving lifecycle owned by the receiving domain**, distinct from orchestrator issuance. Once a PO is delivered to the receiving surface its terminal state is **`RECEIVED`** (not `ISSUED`), and a buyer transitions it via two typed actions, each persisting timestamp + reason:

- **acknowledge** → `acknowledged_at` (+ optional `notes`)
- **reject** → `rejected_at` + `rejection_reason`

Every PO carries a typed **`Trace`** (`requisition_id` / `negotiation_ids` / `award_id`) materialized on the order row, linking it back to its origin for O(1) auditable navigation. The typed fields are carried end-to-end — DynamoDB normalizer → domain model → API → SPA — so no receiving datum is write-only.

## Alternatives Considered

- **`ISSUED` is terminal; no receiving lifecycle.** Rejected: a delivered PO with no downstream state cannot model buyer acknowledgment/rejection and leaves the AD-76 trace chain unusable from the order side.
- **Model ack/reject as free-form status strings.** Rejected: untyped strings without reason/timestamp fields are unauditable and become write-only dead data (the defect PR #95 fixed); typed transitions are auditable and renderable.
- **Recompute the origin trace by scanning negotiations/awards on read.** Rejected: a stored typed `Trace` on the order row is O(1) and stable; a scan is costly and fragile.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Buyer-facing receiving actions (ack/reject with reason + timestamp) are first-class and auditable | The orchestrator's "`ISSUED` = done" assumption had to be corrected across e2e assertions (PR #87) |
| `RECEIVED` is the true terminal state, so KPIs and tests assert the delivered-and-received state, and the origin trace is navigable from the PO | The receiving domain adds fields every reader (normalizer, model, UI) must carry end-to-end, or they silently become dead data (PR #95) |

## Results

Realized in the `test_tenant_app` backend (`dynamo_client.acknowledge_order` / `reject_order` + `_normalize_received_order` surfacing `acknowledged_at`/`rejected_at`/`rejection_reason`/`trace`; the `PurchaseOrder` model + typed `Trace`; `api/orders.py` `POST /api/orders/{id}/acknowledge` and `/reject`) and the PO Inbox frontend (amber highlight + status badges on unacknowledged `RECEIVED` POs, ack/reject controls, clickable trace-chain card). Reads go direct-DynamoDB against `{env}-orders` (`pk`/`sk`). Picks up exactly where AD-97 leaves off (issuance → delivery handoff → receiving), treats AD-9's one-PO-per-supplier as the receiving unit, and materializes the AD-76 identity chain as the `Trace`. Shipped across PR #92 (ack/reject + trace UI), PR #95 (reason wired end-to-end + typed `Trace`), and the terminal-state correction in PR #87.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
