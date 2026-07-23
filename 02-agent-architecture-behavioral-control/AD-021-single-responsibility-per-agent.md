# AD-021 — Single Responsibility per Agent

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-21 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-4, AD-22, AD-27, AD-28, AD-39, AD-57

## Context

The negotiation workflow spans seven distinct cognitive jobs: classify a category, run one of four negotiation strategies, evaluate bids, and write supplier communications. The Kraljic 2×2 routing primitive (AD-5) already produces four mutually exclusive strategy branches. The system could collapse all of this into one large agent with a long system prompt and a wide tool belt, or decompose it into single-domain agents deployed at their respective workflow nodes.

## Decision

Six agents, each owning exactly one cognitive domain, each independently deployable as its own AgentCore Runtime invoked by the Step Functions Orchestrator at a defined node: Kraljic Classifier (Step 2), Spot Bidding (Step 4a), Leverage Auction (Step 4b), Bottleneck Negotiation (Step 4c), Strategic Partnership (Step 4d), and Award & Communications (Step 7). The boundary is a design rule — the Spot Bidding Agent does not evaluate bids, and the Bid Evaluation Agent does not send emails.

## Alternatives Considered

- **Single large multi-skill agent.** Rejected: a wide tool belt and long system prompt couples release cycles, removes per-agent model tiering, and prevents independent testability or blast-radius isolation.
- **Two-agent split (strategy vs. communications).** Rejected: strategy still spans four incompatible quadrants with different tool sets; a single strategy agent would require a large conditional prompt and no Cedar surface could be scoped narrowly enough.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Each agent is independently testable, versionable, and replaceable | Seven runtimes to deploy, monitor, and keep on compatible SDK versions |
| Per-agent model tiering becomes possible — `SimpleLLM` for cheap structured work (Kraljic, Spot, Award), `DefaultLLM` for strategic reasoning; ~3× saving on Scenario A, up to ~13–23× on Scenario B | Coordination cost moves to the orchestration layer; no agent can call another directly |
| A narrow Cedar permit/forbid surface per agent and a bounded blast radius for any single regression | Cross-agent invariants must be enforced outside the agent, at the node or hook level |
| One cognitive domain per prompt keeps each system prompt small — load-bearing for cache economics (AD-28) | State must be threaded through Step Functions shared state rather than held in one process (AD-27) |

## Results

This decision is what makes the cost architecture (AD-57, model tiering by cognitive demand) and the per-agent Cedar least-privilege table (AD-39) expressible — both are stated per agent. A quality regression or a bad deploy is contained to a single Runtime and a single Kraljic branch rather than the whole pipeline. The operational cost is seven Runtimes, seven Agent Cards, and the discipline of never letting responsibilities leak across the boundary.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
