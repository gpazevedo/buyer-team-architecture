# AD-014 — Idempotent Nodes with Explicit Dedup Keys + Checkpoint After Every Node

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-14 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-11, AD-15, AD-12, AD-26, AD-2

> **Correction 2026-08-04 (impl PR #257) — the checkpoint half of this decision was never wired.** `save_checkpoint()` existed in `orchestrator/resilience/checkpoint.py` from the beginning, but had **zero callers anywhere in the repository**: no node ever wrote a checkpoint, the checkpoint table stayed empty, and everything the Results section below claimed about resuming from the last completed node was unreachable. See the dated note in Results for what was actually true and what PR #257 changed.

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

**Correction 2026-08-04 (impl PR #257): the paragraph above described the design, not the deployed system — for the whole period preceding this PR, only half of this decision was real.**

**The dedup-key half was real throughout.** Every dedup key named in the Decision is in the node code and doing its job. That is why the missing half produced no visible corruption and survived this long: idempotency made re-execution safe, so the absence of checkpoints degraded recovery from *resume* to *replay-from-the-top* rather than to duplicate POs.

**The checkpoint half was dead code.** `save_checkpoint()` had no caller outside its own tests (`git grep save_checkpoint 01f44c2^ -- '*.py'` returns the definition and nothing else), so no checkpoint was ever written. Two consumers silently degraded as a result, both without error:

- `resilience/recovery.py::run_recovery_flow` had nothing to resume from, so the recovery sweep it advertises never restored a negotiation.
- `redrive_dlq` escalated **every** redriven message straight to REQUIRES_ATTENTION with `no_checkpoint` — the "operators can resume from checkpoint and reuse partial work" outcome claimed in AD-26 was unreachable by construction.

**What PR #257 changed.** A new `graph_common.checkpoint_node(node_name, result)` is called from the `lambda_handler` of all six node Lambdas (which cover the seven DAG nodes — `node_strategy_execute.py` is Node 3 + Node 4x). It records `node_status="completed"`, or `"interrupted"` when the node returns `PENDING_APPROVAL`. Two deliberate properties: the write is best-effort, wrapped so a checkpoint failure logs a warning rather than failing an already-computed node result; and it skips silently when the result carries no `tenant_id`/`negotiation_id`, which is an early-error return that never reached a negotiation. Wiring it also exposed a latent crash in `save_checkpoint` itself — a bare `json.dumps` over a `graph_state` carrying a `Decimal` (e.g. `budget_limit`) raised `TypeError`; now serialized with `default=str`.

**Still unverified:** the REQ-G300 30-second resume target. The mechanism now exists and is unit-tested, but no live recovery of a real interrupted negotiation has been exercised end-to-end against dev, so the number in the Trade-offs table remains a design target rather than a measurement.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
