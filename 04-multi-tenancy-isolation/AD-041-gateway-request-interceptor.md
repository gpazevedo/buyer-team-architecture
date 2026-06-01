# AD-041 — Gateway Request Interceptor Rewrites tenant_id, Fails Closed

**Theme:** Multi-Tenancy & Isolation  **Catalog:** AD-41 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-37, AD-38, AD-39, AD-42

## Context

Cedar decides whether an agent→tool binding is permitted, but the `tenant_id` argument in a `tools/call` still originates from the agent's reasoning. A prompt-injected or buggy agent operating under tenant B's JWT could emit `tenant_id=A` in the call arguments, and Cedar — evaluating the policy principal, not the argument value — would not catch the mismatch. This is the confused-deputy class of bug: a legitimate agent identity being used to act on behalf of the wrong tenant.

## Decision

A Lambda interceptor on every Gateway decodes the Layer-2-validated JWT, extracts the normalized `tenantId` claim (produced by AD-42), and overwrites `params.arguments.tenant_id` in the request body before Cedar evaluates it. A missing or empty claim returns 403; any interceptor failure returns 500 and the target Lambda never executes (fail closed).

## Alternatives Considered

- **Cedar principal-condition on tenant_id argument.** Rejected: Cedar evaluates the request's principal and context attributes — it cannot reliably enforce that an agent-supplied argument value matches the JWT-bound tenant without a trusted rewrite upstream.
- **Plugin-side argument validation.** Rejected: enforcement inside the Plugin shares the Plugin's failure surface; a compromised or misconfigured Plugin cannot be trusted to enforce its own tenant boundary.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The value Cedar evaluates and the value the Plugin receives is always the JWT-bound tenant, never an agent-supplied one — the confused-deputy bug is structurally closed | A mandatory Lambda hop on every `tools/call`; interceptor availability becomes a hard dependency for all Gateway-routed tool calls |
| The interceptor is a simple deterministic transform (decode JWT → overwrite field), minimizing the complexity that could itself introduce bugs | Interceptor failure mode is 500 for all tool calls (fail-closed design): a broken interceptor blocks all agents until resolved |

SDK-transport Plugins are exempt because they have no Gateway hop.

## Results

The interceptor executes before Cedar so tenant-conditioned permit rules see the trusted value. It emits `security.interceptor.missing_claim` and `security.interceptor.failure` metrics for alerting. Correct operation depends on the consistent `tenantId` claim shape produced by AD-42. This control is the structural fix for the confused-deputy gap identified in the AD-38 four-layer stack.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
