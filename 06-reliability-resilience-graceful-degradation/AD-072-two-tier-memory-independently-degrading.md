# AD-072 — Two-Tier Memory: AgentCore Memory + Mem0, Independently Degrading

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-72 · **Source PRD:** PRD-014 · **Status:** Accepted · **Related:** AD-3, AD-20, AD-46, AD-74, AD-129

## Context

AgentCore Memory is scoped to `(actor_id, session_id)` — it answers "what happened in this conversation" but cannot answer questions that span negotiations: a supplier's history across deals, cross-negotiation bid patterns, an approver's learned preferences, or which communication strategies worked before. Those stubs currently return `[]`, degrading the strategic agents' decisions. AD-3 deliberately placed all durable state in DynamoDB and left cross-negotiation knowledge out of AgentCore Memory; that gap is real and must be filled without overloading session memory with a job it is not designed for.

## Decision

Run two memory tiers side by side. AgentCore Memory handles runtime conversation persistence per `(actor_id, session_id)`; Mem0 is an explicit, tool-called business-knowledge layer for cross-negotiation facts. Each tier degrades independently — neither depends on the other. The feature is controlled by the `mem0_enabled` flag in `{env}-system-config` (default false).

## Alternatives Considered

- **Force cross-negotiation facts into AgentCore Memory's `(actor, session)` scope.** Rejected: the scope does not fit — session memory does not survive across invocations or negotiations, and would require per-call context rehydration of an ever-growing fact set.
- **Store cross-negotiation facts in bespoke DynamoDB tables.** Rejected: heavier to build and operate, and DynamoDB does not provide the semantic search that makes Mem0 useful for retrieving relevant historical context.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Cross-negotiation knowledge that AgentCore Memory's scope structurally cannot provide | A single unified memory abstraction — agents must reason about two stores with different scopes, access patterns, and failure modes |
| Independent degradation: a Mem0 outage does not touch in-session conversation memory, and vice versa | A new external platform dependency (Mem0) and its networking requirement (NAT/VPC endpoint for outbound HTTPS) |
| Fills the AD-3 gap without reversing the deliberate choice to keep DynamoDB as the authoritative record | More surface to understand and operate; two adapters, two circuit breakers, two sets of failure metrics |

## Results

Clear division of labour: "this conversation" (AgentCore Memory) vs "across all conversations" (Mem0). The `memory_degraded` boolean is the OR of the two tiers' degradation signals (AD-20). The four Mem0 integration points — supplier relationship history, cross-negotiation bid patterns, approver preferences, communication effectiveness — are each defined in AD-74 and each degrade gracefully per AD-46. Mem0 is feature-flagged (`mem0_enabled`, default false) and, as of 2026-07-22, still entirely unwired — no Mem0 call site exists anywhere in the agents. IP-1 (supplier relationship history) shipped ahead of Mem0 itself via a direct DynamoDB substitute instead (AD-129); that substitute sits outside this two-tier model rather than validating it. This fills the gap deliberately created by AD-3 without reversing it, but the two-tier design remains aspirational for all four IPs until Mem0 is actually wired.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
