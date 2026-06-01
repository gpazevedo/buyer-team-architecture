# AD-087 — SUSPENDED: Reject New Admissions, Let In-Flight Drain (Optional Force-Cancel)

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-87 · **Source PRD:** PRD-017 · **Status:** Accepted · **Related:** AD-85, AD-81

## Context

Billing holds, compliance reviews, and operational quarantine require a reversible pause that does not destroy infrastructure or data — distinct from decommissioning. When a tenant transitions to `SUSPENDED` (AD-85), the question is what happens to negotiations already in flight: stopping them immediately risks destroying work-in-progress for a routine billing hold, but leaving them running is unacceptable in a compliance suspension requiring an immediate stop.

## Decision

On `SUSPENDED`, reject all new admission requests with reason `tenant_suspended` (PRD-016 REQ-G519). Let existing in-flight negotiations continue to their terminal status by default. Provide an optional `drain_in_flight = true` parameter that force-cancels all non-terminal negotiations and fires the standard counter decrements. Resume (`SUSPENDED → ACTIVE`) requires no dry-run re-run — all artifacts remain in place.

## Alternatives Considered

- **Always force-cancel in-flight negotiations on suspension.** Rejected: destroys work-in-progress for routine billing holds where the only intent is to block new work; the cost and disruption are disproportionate.
- **Always let in-flight drain with no force-cancel option.** Rejected: a compliance suspension may legally require an immediate stop; no force-cancel means the operator has no mechanism to achieve it without a direct database intervention.
- **Block new admissions and block in-flight at their next node boundary.** Rejected: node-boundary blocking is complex to implement and still does not provide an immediate stop; drain mode with explicit cancellation is simpler and more decisive.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Default drain-to-terminal avoids destroying in-progress work for routine billing holds | A suspended tenant running default (no drain) still consumes capacity and incurs cost until in-flight negotiations terminate |
| Resume is a simple state transition — no re-validation, no re-provisioning | Two suspension behaviors (default vs drain) is more API surface than a single behavior |
| Drain mode provides an explicit, auditable mechanism for compliance-required immediate stops | Drain requires the Orchestrator to cancel each non-terminal negotiation individually, with counter decrements cascading |

## Results

New admission rejection reason `tenant_suspended` is distinct from quota-exhausted reasons, aiding client error handling (PRD-016 REQ-G519). Drain emits `tenant.drain.requested` and the Orchestrator cancels non-terminal negotiations with counter decrements cascading (PRD-016 REQ-G510). Resume requires no dry-run re-run (contrast with onboarding, AD-86). The `SUSPENDED` state and its admission behavior are read by AD-81's `ConditionCheck` via the `{env}-tenants` schema owned by AD-85.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
