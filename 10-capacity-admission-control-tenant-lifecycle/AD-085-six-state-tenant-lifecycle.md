# AD-085 — Six-State Tenant Lifecycle; PURGED Retained as a Tombstone

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-85 · **Source PRD:** PRD-017 · **Status:** Accepted · **Related:** AD-81, AD-86, AD-87

## Context

Before PRD-017, the platform described only the tenant create path — there was no defined suspension, decommissioning, or purge, and no authoritative schema for the `{env}-tenants` catalog row that admission control (AD-81) and the reconciler (AD-84) already depend on. Without a defined purge, there is no protection against `tenant_id` reuse collisions after a tenant leaves, and "deleted" has no implementation meaning.

## Decision

Define a six-state machine: `PENDING → ACTIVE → SUSPENDED / DECOMMISSIONING → DECOMMISSIONED → PURGED`. After purge, retain the `{env}-tenants` row as a tombstone containing only `tenant_id`, `status`, `status_transitioned_at`, and `display_name` — all business data deleted, the ID reserved permanently. All per-tenant business data is retained for a 90-day audit window in `DECOMMISSIONED` before purge.

## Alternatives Considered

- **Simple active/deleted flag.** Rejected: provides no defined suspension state, no audit-retention window, and no `tenant_id` reservation after deletion — allowing ID reuse collisions.
- **Hard delete on decommission (no tombstone, no retention window).** Rejected: regulatory and audit requirements mandate data retention for a period after a tenant relationship ends; hard deletion violates those requirements.
- **Three-state machine (ACTIVE / SUSPENDED / DELETED).** Rejected: conflates the reversible pause (SUSPENDED) with the irreversible decommission path and provides no defined PURGED state to lock the ID.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Every lifecycle phase has defined admission and in-flight behavior; the `{env}-tenants` schema is now authoritative for the whole platform | Six states plus per-state behavior tables are more lifecycle surface than a simple active/deleted flag |
| Tombstone prevents `tenant_id` reuse collisions and satisfies audit-retention requirements | Tombstones are permanent (if tiny) — the ID reservation has no expiry |
| Each state's admission behavior (reject reasons) maps exactly to the `ConditionCheck` in AD-81 | "Deleted" is a multi-stage, time-delayed process (90-day DECOMMISSIONED window before PURGED), not an instant one |

PRD-017 now owns the `{env}-tenants` schema that PRD-016, PRD-009 (AD-61), and other PRDs reference. Failed onboarding deletes the row outright — no tombstone, no audit-retention rationale, since ACTIVE was never reached.

## Results

Transitions recorded as `status`, `status_transitioned_at`, `status_transition_reason` and CloudTrail-audited. The cost-attribution table (AD-61) is exempt from purge — financial records age out on their own 397-day TTL. The state machine is realized by the onboarding Step Function (AD-86) for the create path and by the suspension behavior (AD-87) for the reversible pause.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
