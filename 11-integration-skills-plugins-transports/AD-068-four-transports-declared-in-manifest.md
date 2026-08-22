# AD-068 — Four Transports Declared in Manifest, Preference SDK > Kafka > MCP > API

**Theme:** Integration (Skills, Plugins & Transports)  **Catalog:** AD-68 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-67, AD-69, AD-70

## Context

External systems speak different protocols: AWS-native services (SDK), event streams (Kafka), MCP-native tool calls, and plain REST/webhooks. Without a declared transport model, individual integrations would choose transports ad hoc at the infrastructure level, producing an inconsistent surface and making governance of the Gateway-bypass cases (AD-69) invisible. When an external system supports multiple transports, there must be a principled preference order rather than a per-engineer judgment call.

## Decision

Support four transports — MCP, Kafka, REST/Webhook, and AWS SDK — always declared in the Plugin manifest and implemented in the Skill. The preference order when multiple transports are available is: SDK > Kafka > MCP > API. SDK is preferred for AWS-native integrations (IAM-bound, no Gateway hop); among the remaining three, Kafka > MCP > API favors decoupling and resilience over synchronous coupling. A single integration domain may declare multiple transports (e.g., inbound via Kafka, outbound confirmation via API). Transport is immutable at runtime; changes require a Plugin manifest update and re-registration.

## Alternatives Considered

- **Transport chosen ad hoc per integration.** Rejected: produces an ungoverned mix of gateway and non-gateway paths, making the enforcement asymmetry in AD-69 invisible and unauditable.
- **Single universal transport (MCP only).** Rejected: excludes high-volume async use cases naturally served by Kafka, and forces AWS-native integrations through an unnecessary Gateway hop.
- **Preference order API > MCP > Kafka > SDK (synchronous-first).** Rejected: favors tight coupling over resilience; the most-preferred transport would be the most fragile, and AWS-native integrations would pay an unnecessary Gateway hop.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One integration model covers every external system shape; the manifest is the single source of truth for transport choice | Uniformity of enforcement: SDK and Kafka bypass the Gateway (AD-69), so the most-preferred transports skip Cedar and the tenant-id Interceptor |
| Preference for decoupling (SDK, Kafka) reduces Gateway hop overhead and increases resilience for high-volume paths | Each bypassing transport requires its own compensating isolation mechanism (IAM role for SDK; topic naming + broker ACLs for Kafka), creating non-uniform tenancy enforcement |
| A domain may combine transports (e.g., inbound Kafka + outbound API) without a code change to the Skill | Transport immutability at runtime means a protocol change requires a new manifest and re-registration — not a live reconfiguration |

The enforcement asymmetry is a direct consequence of the preference order: the most-preferred transports are exactly those that skip the Gateway's controls. This asymmetry is explicit and documented; the compensating controls per transport are defined in AD-69.

## Results

**Decision accepted; the manifest mechanism is not built.** As of 2026-08-20 the Plugin manifest does not exist in the implementation — there is no manifest schema, no `sdk_config.resource_arns` declaration, no deploy-time transport validation, and no Terraform plan-time immutability check. Of the four transports, only MCP and SDK have code: MCP through the ingest and receiving AgentCore Gateways (`infra/gateway.tf`, `infra/receiving_gateway.tf`), SDK through the Skill Runtime's own IAM role. **Kafka has no implementation at all** — no client, no topics, no MSK — so the topic-ACL tenancy mechanism this ADR names is likewise unbuilt (PRD-011 §10 phases it at Phase 3/4). REST/webhook exists only as the SES email leg of supplier comms.

Consequently the enforcement asymmetry this ADR trades away is currently *theoretical*: every live integration path goes through the Gateway's Cedar policy engine and tenant-id interceptor, or is IAM-bound SDK. The asymmetry becomes real the moment Kafka ships, and the compensating controls in AD-69 must land with it rather than after.

What the decision does bind today is the preference order and the requirement that transport be declared rather than chosen ad hoc — honoured by convention, not by a validator. Re-verify this section against `impl/` before citing it as as-built.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
