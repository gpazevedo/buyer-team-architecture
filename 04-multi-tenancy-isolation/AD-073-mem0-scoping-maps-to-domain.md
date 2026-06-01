# AD-073 — Mem0 Scoping Maps to the Domain Model

**Theme:** Multi-Tenancy & Isolation  **Catalog:** AD-73 · **Source PRD:** PRD-014 · **Status:** Accepted · **Related:** AD-38, AD-72

## Context

Mem0's generic scoping fields (`user_id`, `agent_id`, `run_id`, `app_id`) are not pre-mapped to any domain concept. Without a consistent convention, memory queries could cross tenant boundaries or lose attribution, since Mem0 has no built-in awareness of the Buyer Team domain model. Mem0 holds enhancement data (supplier history, bid patterns, approver preferences, comms effectiveness) that feeds into negotiations — cross-tenant leakage here would violate the invariant in AD-38, even though the data is not the authoritative record.

## Decision

Map Mem0 scoping directly onto the domain: `user_id` = `tenant_id`, `agent_id` = agent node name, `run_id` = `negotiation_id`, `app_id` = `"buyer-team"`. Every Mem0 read and write must set these fields consistently; the mapping table is the single authoritative source for the convention.

## Alternatives Considered

- **IAM/partition-key isolation inside Mem0.** Rejected: Mem0 does not expose IAM-level access control; the scoping API is the only isolation mechanism available and is appropriate given that Mem0 holds enhancement data, not authoritative records.
- **Omit tenant scoping; rely on `run_id` alone.** Rejected: `negotiation_id` alone does not prevent cross-tenant queries if `user_id` is left unset; a tenant filter must be the outermost scope on every read.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Tenant isolation in Mem0 at no extra infrastructure cost — every read and write is filtered to `user_id=tenant_id` by the scoping convention | Coupling to Mem0's scoping semantics: the isolation guarantee depends on every call site correctly setting `user_id=tenant_id`, since Mem0 itself has no tenant concept |
| Agent attribution (`agent_id` = node name) and negotiation traceability (`run_id` = negotiation_id) come for free from the same mapping | The isolation is weaker than the IAM and partition-key controls used for primary data — intentionally, because Mem0 holds enhancement data, not the authoritative record |

## Results

Tenant-filtered semantic search is enabled by default (e.g., supplier delivery-issue history is scoped to the requesting tenant). Memory writes carry node and negotiation attribution through `agent_id` and `run_id`. The mapping table is the single source of the convention and must be kept in sync with any new agent nodes. This isolation is deliberately weaker than the four-layer stack (AD-38) applied to primary data — the weakness is proportional to what the store holds. The Mem0 integration is feature-flagged (`mem0_enabled`, default false) and integrates with the two-tier memory model established by AD-72.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
