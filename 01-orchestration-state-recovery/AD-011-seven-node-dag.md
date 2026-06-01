# AD-011 — Seven-Node DAG with a Single Four-Way Branch

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-11 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-1, AD-5, AD-18

## Context

Procurement negotiation has to satisfy three constraints that a free-running multi-agent system cannot meet: regulatory auditability requires a deterministic, traceable execution path; governance must be enforced structurally rather than through prompt text; and failure recovery must resume from known checkpoints, not restart the whole workflow. The four procurement strategies fall out of the Kraljic 2×2 matrix (profit impact × supply risk), so the routing surface is naturally finite.

## Decision

Model every negotiation as a fixed seven-node directed graph. A single four-way conditional branch sits at Node 3 (Strategy Router), keyed solely on the Kraljic quadrant. Exactly one Node 4 variant (4a Spot Bid, 4b Leverage Auction, 4c Bottleneck, 4d Strategic) executes per negotiation; all four converge at Node 5 (Bid Evaluation) and the graph terminates at Node 7 (Award & Communications). Routing is deterministic local Python with no LLM involvement (REQ-G100, Node 3 < 100ms per REQ-G401).

## Alternatives Considered

- **Fully autonomous agent swarm.** Rejected: no deterministic path to audit, governance only expressible as prompt guidance, recovery means full restart.
- **One independent graph per strategy.** Rejected: duplicates the shared head (Ingest, Classify) and tail (Evaluate, Approve, Award), and loses the single converging evaluation point.
- **Deeper / nested branching.** Rejected: the Kraljic matrix already collapses the decision space to four quadrants; extra branching adds maintenance with no routing gain.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Deterministic, auditable execution path with a single source of workflow truth | Strategy set is fixed at four — a fifth negotiation style requires a graph change plus a new Node 4 variant |
| Node 3 routing is sub-100ms pure Python with no LLM involvement | All four Node 4 variants must produce bids in the shape Node 5 understands, constraining how divergent they can be |
| Fixed shape makes checkpoint-based recovery and governance-in-code tractable | |

## Results

Every negotiation has a known, replayable path, which makes flow-adherence a code-based evaluation (all expected node spans present, in order) and routing a ground-truth evaluation against the Kraljic dataset. Node 3 stays sub-100ms because it is pure mapping. The fixed shape is also what makes checkpoint-based recovery (AD-14) and the governance-in-code model (AD-18) tractable.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
