# AD-067 — One Skill + One-or-More Plugins per Tenant

**Theme:** Integration (Skills, Plugins & Transports)  **Catalog:** AD-67 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-68, AD-69, AD-70

## Context

Every tenant integrates with different external systems (SAP, Oracle, Coupa, supplier portals) over different protocols, but the integration logic — schema mapping, idempotency, validation — is largely the same regardless of the external ecosystem. Without a clear separation, each new external system would require a new logic implementation, multiplying maintenance surface as both tenant count and external-system count grow. The domain model and DynamoDB tables must never be directly accessible to external systems; a boundary artifact is required at every integration point.

## Decision

One **Skill** per tenant holds all integration logic and tool implementations; one-or-more **Plugins** per tenant are pure transport adapters that declare the transport mechanism and expose the Skill to a specific external ecosystem. Plugins contain no logic — they are packaging, registration, and transport-declaration artifacts. The same Skill serves SAP, Coupa, Oracle, and any future system, configured differently via `{env}-tenant-skill-config`. Plugins register as AgentCore Gateway targets (the Gateway's target/tool model is the registration surface); there is no separate Plugin API.

## Alternatives Considered

- **One implementation per external system.** Rejected: logic duplicates across every ERP/P2P system, making maintenance and testing proportional to the cross-product of tenants and external systems rather than to tenants alone.
- **Logic embedded in the Plugin.** Rejected: prevents reuse of the same logic across transport types; a schema-mapping bug would need to be fixed in every Plugin separately.
- **Status quo / no action.** Rejected: without the Skill/Plugin split, transport and logic are conflated, blocking the multi-transport capability required by AD-68.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A tenant on SAP and a tenant on Coupa use the same Skill, configured differently; integration logic is tested once | Every integration now has two artifacts (Skill + Plugin) and a binding — more structure than a single per-system implementation for a one-off |
| Adding a new external system requires only a new Plugin manifest and transport declaration, not new logic | The binding between Skill and Plugin must be maintained in `{env}-tenant-skill-config` per tenant |
| Logic is independently deployable and versionable from transport adapters | Plugin registration must be repeated per external system even when the Skill is unchanged |

## Results

Per-tenant configuration lives in `{env}-tenant-skill-config` (PK: `tenant_id`, SK: `skill_config`). Plugins register as AgentCore Gateway targets for MCP, Kafka, and API transports; SDK-transport Plugins bypass the Gateway (the IAM role is the registration). This split is the structural prerequisite for the four-transport model in AD-68 and the Gateway-bypass pattern in AD-69.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
