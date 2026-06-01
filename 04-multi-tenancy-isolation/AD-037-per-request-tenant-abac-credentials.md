# AD-037 — Per-Request Tenant ABAC Credentials

**Theme:** Multi-Tenancy & Isolation  **Catalog:** AD-37 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-6, AD-38, AD-41, AD-42

## Context

Per-Runtime IAM bounds what kinds of operations an agent can do, and per-tenant Skill Runtimes give each tenant a separate address — but neither prevents a single Runtime acting for tenant A from touching tenant B's data on a shared Plugin. Application-layer checks in Plugin code were the only prior defence, leaving cross-tenant access open if Plugin code is buggy or compromised. The gap needs to be closed at the IAM layer so it cannot be bypassed by application code.

## Decision

The Gateway Interceptor assumes a per-Gateway ABAC role with `tenant_id` injected as an STS session tag (`sts:TagSession` + `${aws:PrincipalTag/tenant_id}` ARN substitution) and forwards the scoped temporary credentials to the target Plugin. The Plugin's effective IAM permissions are bounded per request by the session-tagged tenant identity. This pattern is adopted from the `aws-samples/sample-saas-multi-agents-workshop` isolation pattern, GA in v1.0 after a v1.5.2 fast-follow period.

## Alternatives Considered

- **Application-layer Plugin checks only.** Rejected: buggy or compromised Plugin code can bypass application-level tenant checks; an IAM-layer control is needed that holds independent of Plugin correctness.
- **Per-tenant Runtimes with no shared Plugins.** Rejected: prohibitively expensive and operationally complex at scale; the ABAC pattern achieves equivalent IAM isolation without duplicating infrastructure per tenant.

## Trade-offs

| Gained | Given up |
| --- | --- |
| IAM-layer cross-tenant protection even if Plugin code is buggy or compromised — the storage engine denies the request unless the resource ARN matches the session-tagged tenant | An extra `AssumeRole` per request adds latency and introduces an `assume_role_failure` failure mode that must fail closed |
| Scoped temporary credentials reduce the blast radius of any credential leak to a single tenant and a single request window | Applicability is limited to Plugins whose ARNs are tenant-partitioned; non-partitioned targets (`email`, outbound ERP HTTP) must rely on other layers |

SDK-transport Plugins are explicitly excluded because they have no Gateway hop; they inherit isolation from the Skill Runtime IAM role.

## Results

Five per-Gateway ABAC roles are provisioned, with an `abac_tools` registry in system-config enumerating which tools use ABAC-scoped credentials. A `security.abac.assume_role_failure` alarm fires on credential-assumption failures. This layer sits above the partition-key layer (AD-6) and below Cedar (AD-39) in the four-layer stack anchored by AD-38.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
