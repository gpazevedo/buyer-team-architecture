# AD-069 — Kafka and SDK Transports Bypass the Gateway

**Theme:** Integration (Skills, Plugins & Transports)  **Catalog:** AD-69 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-67, AD-68, AD-70, AD-71

## Context

The AgentCore Gateway is the natural enforcement point for Cedar policy (AD-39) and the tenant-id Interceptor (AD-41) — but Kafka brokers and AWS SDK calls are direct connections from the Skill Runtime container. They do not fit the Gateway's MCP request/response model: Kafka is a persistent streaming connection to a broker, and SDK is a `boto3` call bound to the Runtime's IAM role. Forcing these through the Gateway would add an unnecessary hop and require wrapping protocols that are not natively MCP. Without a clear decision, the bypass would happen implicitly and unaudited.

## Decision

Make the Gateway path MCP-only. Kafka and SDK transports connect directly from the Skill Runtime container — Kafka to the broker (AWS MSK IAM auth, Confluent API key, or self-managed SASL/TLS), SDK via `boto3` with the Runtime's IAM role — bypassing the Gateway entirely. Isolation is re-established per transport: SDK Plugins rely on the Skill Runtime IAM role (every `sdk_config.resource_arns` value must be covered by the role, validated at deploy time); Kafka relies on the topic naming convention `{env}.{tenant_id}.{domain}.{direction}` plus broker ACLs.

## Alternatives Considered

- **Route all transports through the Gateway.** Rejected: requires wrapping Kafka streams and SDK calls in MCP envelopes, adding latency and complexity to protocols that have no natural request/response shape.
- **Apply Cedar at the Kafka broker directly.** Rejected: Cedar is not supported natively on Kafka brokers; enforcement would require a custom interceptor layer not available in the platform.
- **No compensating controls for bypassing transports.** Rejected: leaves Kafka and SDK paths without tenant isolation enforcement, violating the defense-in-depth invariant (AD-38).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Direct, efficient connections for high-volume Kafka streams and AWS-native SDK calls — no Gateway hop, no MCP wrapping of natively non-MCP protocols | The Gateway's centralized enforcement (Cedar, tenant-id Interceptor, per-request ABAC) does not apply to Kafka or SDK paths |
| SDK registration IS the IAM role — no manifest-level credential exchange, no separate adapter | MCP traffic and Kafka/SDK traffic receive different sets of controls, creating an explicit asymmetry that must be maintained per transport |
| Kafka broker-agnostic client configured from `{env}-tenant-skill-config` — no code change when a tenant switches broker implementations | SDK ARN coverage must be validated at deploy time (REQ-M510 / REQ-I108); a validation miss would leave an ARN uncontrolled |

The bypass is an explicit design choice, not an oversight. It is documented so that new transport integrations are evaluated against the same isolation requirement — and so that AD-70 (predicate rewriting) and AD-71 (claims ceiling) are understood as compensating controls for the enforcement gap, not standalone features.

## Results

SDK manifests are validated at deploy time against the Runtime IAM role (every `sdk_config.resource_arns` value must be covered; REQ-M510 / REQ-I108). Kafka topic convention `{env}.{tenant_id}.{domain}.{direction}`; broker client and auth resolved from `{env}-tenant-skill-config` at runtime. The bypass creates the enforcement gap that AD-70's predicate rewriting closes at the query layer, and that AD-71's claims ceiling closes at the ingestion boundary.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
