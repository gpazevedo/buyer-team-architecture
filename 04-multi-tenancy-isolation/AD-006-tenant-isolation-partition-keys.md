# AD-006 — Tenant Isolation via Partition Keys + JWT Claim + Context Injection

**Theme:** Multi-Tenancy & Isolation  **Catalog:** AD-6 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-37, AD-38, AD-39, AD-42

## Context

Buyer Team is multi-tenant SaaS. Cross-tenant data exposure is the highest-severity failure class. LLM prompts cannot be trusted to enforce isolation — they can be ignored, forgotten, or subverted by injection. Without a structural, engine-level isolation mechanism, any prompt-injected or confused agent could read or write another tenant's data. A purely application-layer check is vulnerable to the same failure modes as the model itself.

## Decision

Enforce isolation at the data engine, not in prompts: DynamoDB partition keys are namespaced `tenant_id#…`, requests carry a `tenantId` JWT claim, and tenant context is injected into the execution environment rather than supplied by the model. The database engine, not the LLM, decides what a tenant can see.

## Alternatives Considered

- **Prompt-based isolation.** Rejected: the LLM is an untrusted enforcement surface — injection, hallucination, or context loss can bypass any prompt-carried restriction.
- **Application-layer-only checks.** Rejected: code-layer checks share the failure surface of the application; a buggy or compromised agent can bypass them, providing no structural guarantee.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Isolation holds even under prompt injection or a confused agent — it is structural, enforced by the database engine | Schema freedom: every table and query must carry `tenant_id`; key design is permanently constrained by tenant namespacing |
| Tenancy is a key-design property of every table, making accidental cross-tenant reads structurally impossible at the storage layer | Partition-key isolation alone is not sufficient; additional layers (ABAC, Cedar, predicate rewriting) are required to cover non-partitioned targets and integration paths |

## Results

This is the innermost layer of the four-layer defense-in-depth stack declared in AD-38. Per-request ABAC with a `tenant_id` STS session tag (AD-37) supplements it at the IAM layer; Cedar policy enforcement (AD-39) supplements it at the tool-access layer; tenant-predicate rewriting at integration Plugins (AD-70) closes the integration-layer gap. No single layer's failure produces cross-tenant exposure. The `tenantId` claim shape is normalized for both user and machine-to-machine tokens by AD-42, ensuring the JWT binding that backs partition-key access control is consistent and non-spoofable.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
