# AD-119 — Per-Tenant Cognito M2M App Client for PR Event Router

**Theme:** Multi-Tenancy & Isolation
**Catalog:** AD-119 · **Source PRD:** PRD-007, PRD-005 · **Status:** Accepted · **Related:** AD-006, AD-042, AD-094

## Context

The PR Event Router consumes Cognito M2M client-credentials tokens to call the ingest Gateway. Before this decision, all tenants shared a single `pr_event_router` M2M App Client. The pre-token-generation Lambda V3 normaliser (AD-042) reads the App Client's metadata to inject the `tenantId` claim — with a single shared client, every tenant's PR minted a JWT claiming the same tenant identity (effectively the test-tenant). This meant Blue Jets PRs (a separate demo tenant) carried `tenantId=test_tenant` on their JWTs, breaking tenant isolation at the Gateway interceptor (AD-041) which rewrites `tenant_id` from the JWT claim.

The fix is a dedicated M2M App Client per tenant so the pre-token normaliser can inject the correct per-tenant `tenantId` claim based on which client authenticated. The router selects the right client by tenant_id from a `COGNITO_CLIENT_MAP` environment variable.

## Decision

Provision a second Cognito M2M App Client (`pr_event_router_blue_jets`) alongside the existing test-tenant client. The PR event router reads `COGNITO_CLIENT_MAP` (JSON: `{tenant_id: client_id}`) and selects the client per-request by the incoming PR's `tenant_id`. Token caches and client-secret caches are per-client-id to prevent cross-tenant token collision in warm invocations. The ingest Gateway's `allowed_clients` list admits both router clients. A tenant binding row in `{env}-tenants` with the `by-app-client` GSI maps the Blue Jets client to its tenant UUID (`44c2bf31-31d6-56f5-89da-59d0a697d606`, uuid5 of `root:tenant:blue-jets`).

Backward compatible: an empty or absent `COGNITO_CLIENT_MAP` causes a clear `RuntimeError` at startup. The in-process boto3 fallback path (direct ingest, Gateway bypass) is unchanged.

## Alternatives Considered

- **Multi-tenant single client with a custom claim parameter.** Rejected: Cognito M2M client credentials flow has no user interaction and no way to pass a per-request tenant hint — the `client_credentials` grant produces a token scoped to the client's own metadata only.
- **Separate Cognito User Pool per tenant.** Rejected: this is the federated IdP path (AD-094), designed for human interactive login per tenant, not for M2M service accounts. M2M clients share the central pool.
- **Have the pre-token normaliser derive tenant from the requested scope.** Rejected: scopes are coarse-grained (`negotiate`, `read`, `approve`), not tenant-specific, and repurposing them as tenant identifiers conflates authorization with identification.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Correct per-tenant `tenantId` claim on every M2M JWT — Blue Jets and test-tenant identities are properly separated at the Gateway interceptor | New M2M App Client per tenant — O(N) clients rather than one shared client; provisioning a new tenant now requires a new Cognito client + tenant binding row + `COGNITO_CLIENT_MAP` update |
| Token/secret caches are per-client, eliminating a cross-tenant cache-collision bug that would have appeared on the first warm invocation serving two tenants | The `COGNITO_CLIENT_MAP` env var must be kept in sync with provisioned clients — an omitted tenant produces a clear `RuntimeError` rather than a silent wrong-tenant JWT |

## Results

Shipped in PR #149 (merged 2026-07-06). 9 files changed (+102/−20). Blue Jets UUID: `44c2bf31-31d6-56f5-89da-59d0a697d606`. 9/9 existing tests pass, `terraform validate` clean. Complements AD-042 (pre-token V3 normaliser) and AD-041 (Gateway interceptor) — together the three ADRs form the complete tenant-identity chain: per-tenant client → normaliser injects correct claim → interceptor enforces it.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
