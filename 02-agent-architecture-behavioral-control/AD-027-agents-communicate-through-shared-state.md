# AD-027 — Agents Communicate Only Through Step Functions Shared State

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-27 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-1, AD-2, AD-3, AD-4, AD-14

## Context

Seven agents must pass data along the pipeline — classification feeds strategy selection, strategy produces bids, bids feed evaluation, and the award feeds communications. They could discover and call each other directly (a mesh), or the Step Functions Orchestrator could mediate all data flow through a shared state dictionary (a hub). The choice interacts with recovery: if an agent holds state that another depends on, a crash requires reconstructing a conversation rather than replaying state.

## Decision

No direct inter-agent communication. Each agent reads its input from the Step Functions shared state dictionary and returns output to be merged back; coordination is entirely the orchestrator's responsibility. Steps 4a–4d are mutually exclusive — exactly one strategy agent runs per negotiation. Concurrency exists only within an agent (e.g., Spot Bidding's parallel `send_bid_invitation` calls, bounded by `max_concurrent_bids` from system-config). Agents hold no cross-session state (REQ-A007).

## Alternatives Considered

- **Direct inter-agent mesh (agents call each other via A2A).** Rejected: agents would need service discovery, would carry coupling to each other's schemas, and a crash mid-conversation would require reconstructing state from multiple agents rather than replaying a single shared-state checkpoint.
- **Shared message bus (agents publish/consume events).** Rejected: introduces ordering and delivery guarantees that must be managed outside Step Functions; the orchestrator already provides a durable, ordered, checkpointed state store — duplicating that with a bus adds complexity without benefit.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Loose coupling — any agent can be replaced, re-versioned, or tested in isolation against a state fixture | All coordination latency routes through the orchestrator; there is no fast agent-to-agent path |
| A single source of workflow truth, and clean checkpoint/recovery because state lives in the orchestrator + DynamoDB (AD-3, AD-14) | The shared-state schema becomes a coupling point every agent depends on; schema changes ripple across all seven agents |
| No agent needs service discovery or another agent's address | Patterns that would be natural as a direct conversation must be modeled as node boundaries |

## Results

This is the agent-side reflection of AD-1 (Orchestration Before Intelligence): it is why agents can be stateless and ephemeral, and why recovery (AD-14) is a matter of replaying state rather than reconstructing a conversation. Each agent's outputs being written to shared state also enables the per-node DynamoDB checkpoint (AD-14) to capture the full pipeline result at every stage. The cost is that the shared-state schema is now load-bearing across all seven agents — a benefit for auditability and isolation, a constraint on how freely the data shape can evolve.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
