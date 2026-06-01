# AD-070 — Tenant Predicate Rewriting + User-Action Claim Propagation

**Theme:** Integration (Skills, Plugins & Transports)  **Catalog:** AD-70 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-38, AD-41, AD-69, AD-71

## Context

A Plugin that accepts agent-generated query strings — PartiQL, Athena SQL, ERP query DSLs — has a residual cross-tenant leak: the agent might emit a query with no `tenant_id` predicate, or one referencing the wrong tenant. Cedar (AD-39) can permit the tool but cannot inspect the query body; ABAC (AD-37) bounds the IAM-visible resource but cannot prevent a within-resource leak when one table holds multiple tenants' rows. The Kafka and SDK transports bypass the Gateway entirely (AD-69), so the Gateway Interceptor's forced `tenant_id` rewrite (AD-41) does not apply to those paths. Without a query-layer control, the defense-in-depth stack (AD-38) has a gap at the integration boundary.

## Decision

At query-bearing Plugins, force-inject the JWT-bound `tenant_id` predicate into agent-generated queries before execution, regardless of what the agent emitted. Propagate user-action claims at the integration boundary so that authorization decisions downstream reflect the original JWT context. This is the integration-layer member of the four-layer tenancy defense (AD-38: partition keys → ABAC → Cedar → predicate rewriting).

## Alternatives Considered

- **Rely on Cedar alone to prevent cross-tenant queries.** Rejected: Cedar operates at the tool level and cannot inspect query bodies; it would allow the tool call while the query itself leaks data.
- **Rely on ABAC alone.** Rejected: ABAC bounds IAM-visible resources but cannot prevent within-resource leaks in tables that hold multiple tenants' rows under a single resource ARN.
- **Sanitize queries in the agent's system prompt.** Rejected: prompt-based controls have measured lower compliance (82.5%) than code-level enforcement (AD-23); a predicate-injection vulnerability in an LLM output cannot be reliably closed by prompting.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Closes the within-resource cross-tenant leak that neither Cedar nor ABAC can reach — the query that executes always carries the JWT-bound tenant predicate | The Plugin must parse and rewrite query bodies, which is transport- and DSL-specific work (PartiQL ≠ Athena SQL ≠ an ERP query DSL) |
| Defense-in-depth completeness: adds a fourth layer at the integration boundary where the Gateway Interceptor cannot reach | Query parsing logic is itself a risk surface — a parsing bug in the rewriter could become a vulnerability |
| User-action claim propagation ensures authorization context is preserved across the integration boundary | Each new query DSL added by a new external system requires a corresponding rewriting implementation |

The DSL-specific parsing cost is accepted because no single upstream layer (Cedar, ABAC, partition keys) can close this gap. The rewriter is part of the defense-in-depth stack (AD-38), not a replacement for any layer within it.

## Results

Predicate rewriting is implemented in query-bearing Plugins; the rewritten `tenant_id` predicate is always sourced from the JWT-bound claim, not from agent output. Pairs with the user-action claim model bounded by AD-71. Together with partition keys (AD-6), ABAC (AD-37), and Cedar (AD-39), this completes the four-layer tenancy defense described in AD-38.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
