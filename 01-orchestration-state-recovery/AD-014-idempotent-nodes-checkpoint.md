# AD-014 — Idempotent Nodes with Explicit Dedup Keys + Checkpoint After Every Node

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-14 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-11, AD-15, AD-12

## Context

"Orchestration Before Intelligence" requires recovery from known checkpoints rather than full-workflow restart. A node may crash after producing side effects (sent invitations, written bids, issued POs); a naive resume could duplicate those effects.

## Decision

Persist state to DynamoDB after every node completion (REQ-G002), and make every node idempotent by detecting previously completed work through explicit dedup keys: `(tenant_id, category_id, hash(items), deadline)` at Node 1, the semantic cache at Node 2, `(negotiation_id, supplier_id, action, round_number)` at Node 4x, an existing `evaluation_score` at Node 5, and existing `CommunicationLog` at Node 7. Tool-level idempotency (AD-26) backs this at the agent layer.

## Alternatives Considered

- **Checkpoint only at major milestones.** Rejected: coarser recovery means re-executing expensive agent calls and risking duplicate side effects.
- **No idempotency, rely on restart-from-scratch.** Rejected: violates the recovery requirement and produces duplicate bids/POs/communications.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Recovery granularity is one node — resume within 30s (REQ-G300) with no duplicate bids, POs, or communications | A DynamoDB write and bookkeeping cost per node, scaling with node count |
| No node can double-apply its side effects on resume | Each dedup key must capture exactly the inputs that define "the same work" — a non-trivial design obligation per node |

## Results

A runtime can resume a negotiation from the last completed node within 30s (REQ-G300) with no duplicate bids, POs, or communications. Cycle-back recovery is safe because re-marking already-SUPERSEDED bids is a no-op. The checkpoint table is also the substrate for the recovery lock (AD-15) and the 30-day audit retention while in REQUIRES_ATTENTION.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
