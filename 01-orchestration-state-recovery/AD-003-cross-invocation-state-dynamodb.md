# AD-003 — Cross-Invocation State in DynamoDB, Not AgentCore Memory

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-3 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-2, AD-72

## Context

AgentCore Memory *can* persist across sessions, so it is technically available as a system of record. But the audit, query, and orchestration requirements need structured, queryable, Step-Functions-readable state — properties an opaque semantic memory store does not provide. This decision resolves the gap forced by AD-2 (ephemeral sessions) by naming an authoritative store.

## Decision

Use AgentCore Memory only for turn-by-turn context *within a single invocation*. All cross-invocation state — negotiation lifecycle, bids, awards, supplier history — lives in DynamoDB. PRD-001 §4.1 states this explicitly as "a design choice, not a platform limitation."

## Alternatives Considered

- **Use AgentCore Memory as the authoritative cross-invocation store.** Rejected: AgentCore Memory is opaque semantic storage scoped to `(actor, session)`; it is not queryable by Step Functions, provides no structured audit record, and cannot represent the 21-table negotiation chain.
- **Use a relational database (RDS/Aurora).** Rejected: adds a managed server-based dependency; DynamoDB's native integration with Step Functions and its serverless scaling are a better fit for event-driven, per-tenant isolation via partition keys.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Structured, queryable state with native Step Functions integration | All cross-invocation data must be explicitly modelled in DynamoDB tables (21 by v1.0.36) |
| Single authoritative source of truth for the full negotiation chain | Out-of-the-box convenience of platform-managed semantic memory across sessions |
| Audit-complete record readable by orchestration, dashboards, and evaluators | AgentCore Memory's cross-session knowledge cannot represent facts that span multiple negotiations |

The gap created by excluding cross-negotiation knowledge from this store is deliberate: it is filled by a separate, independently-degradable subsystem (AD-72) rather than overloading a single mechanism.

## Results

That gap is filled by a separate two-tier memory model in PRD-014 (AD-72): AgentCore Memory for "what happened in this conversation" and Mem0 for "what we know across all conversations." This ADR is therefore the reason cross-negotiation memory is an explicit, independently-degradable subsystem rather than an implicit platform feature.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
