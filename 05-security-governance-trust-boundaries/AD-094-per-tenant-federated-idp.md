# AD-094 — Per-Tenant Federated IdP for Non-Spoofable Human Tenant Binding

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-94 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-42, AD-37, AD-41, AD-6, AD-86, AD-108

## Context

Tenant identity must be a non-spoofable binding, not a request-supplied value (AD-6, AD-42). The M2M path solves this with the authenticated App Client → `by-app-client` GSI lookup (AD-42 / REQ-S708): a `client_credentials` caller cannot assert another tenant's id because the App Client secret *is* the credential. But interactive (human) sign-in had no equivalent binding. If all tenants' users authenticated against one shared Cognito-native pool, the `custom:tenantId` attribute would be set on user records administratively, and any defect in that provisioning — or any path that let a user influence their own attributes — would be a cross-tenant identity forgery. Role/group-based `ClaimRequirement`s (PRD-011 §2.1 `tenant_default_claims` referencing `userRole`/`groups`) were also unsatisfiable for human users, because a Cognito-native pool issued only `tenantId` + OAuth `scope` (the gap flagged in PRD-005 v1.5.2 finding #2). The federation work was first framed as a Phase-2 fast-follow (v1.5.2), then brought to GA in v1.6.0 once PRD-005-impl wired it.

## Decision

Each tenant that uses interactive sign-in gets a **dedicated external IdP (SAML or OIDC) registered as a per-tenant Cognito Identity Provider in the shared user pool**, created at onboarding (REQ-S709; PRD-017 §4.2a / PRD-011-impl §8), with `provider_name == tenant_id`. Three rules make the human tenant binding non-spoofable and the claim shape uniform:

- **REQ-S709** — one IdP per tenant, `provider_name` equals the tenant id.
- **REQ-S710** — the IdP's attribute mapping populates `custom:tenantId` with the tenant id (the binding) and maps IdP role/group attributes to `custom:userRole` / `custom:groups`. Because the IdP is per-tenant and the mapping is fixed at onboarding, a federated user **cannot assert another tenant's id** — non-spoofable in the same sense as the M2M App-Client binding (REQ-S708). The V3 Pre-Token-Generation Lambda (AD-42 / REQ-S707) then promotes these to top-level `tenantId` / `userRole` / `groups` claims.
- **REQ-S711** — each federated tenant has a per-tenant user App Client (`authorization_code`) whose `supported_identity_providers` is exactly that tenant's IdP, so one tenant's login endpoint can authenticate only against that tenant's IdP.

The base/control-plane App Client remains Cognito-native (no federation), preserving a non-federated control-plane sign-in path.

## Alternatives Considered

- **Single shared Cognito-native pool; set `custom:tenantId` administratively per user.** Rejected: the tenant binding would rest on provisioning correctness rather than the authentication path itself — a confused-deputy / identity-forgery surface, and the same single-mechanism dependence AD-6 exists to avoid. (This rejected shape is deliberately adopted, scoped to the demo tenant only, by AD-108 — see the scope note in Results.)
- **One Cognito user pool per tenant (full silo).** Rejected: multiplies pool management, the V3 trigger, and Gateway-authorizer wiring per tenant; the shared-pool + per-tenant-IdP shape keeps one normalization path (AD-42) while still giving a per-tenant trust root.
- **No human federation; M2M / API-key access only.** Rejected: forecloses interactive buyer sign-in and leaves role/group `ClaimRequirement`s permanently unsatisfiable for human approvers (PRD-005 §5.5 entity plane, AD-19).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Non-spoofable human tenant binding rooted in the authentication path, not in attribute provisioning — symmetric with the M2M App-Client binding (AD-42) | An onboarding-time provisioning step per tenant (IdP registration + attribute mapping + dedicated App Client), owned by the onboarding Step Function (AD-86) |
| Role/group claims (`userRole`/`groups`) become satisfiable for human users, enabling fine-grained entity-plane `ClaimRequirement`s (AD-19) | A `tenant_default_claims` entry referencing `userRole`/`groups` is satisfiable only once that tenant's IdP is registered and mapped — a sequencing dependency caught by dry-run check #7 |
| One claim-normalization path (AD-42 V3 Lambda) serves native, federated, and M2M principals identically | SAML/OIDC integration per tenant is external-dependency-bound (the customer's IdP must be reachable and correctly configured) |

The binding is only as strong as the per-tenant IdP registration being correct and immutable post-onboarding; the fixed attribute mapping is what makes cross-tenant assertion impossible, so that mapping is a security-critical onboarding artifact, not a tunable.

## Results

Realized in PRD-005 §4.1 / §5.5.2 and REQ-S709–S711, provisioned by the onboarding flow (PRD-017 §4.2a, PRD-011-impl §8). It completes AD-42: the V3 Pre-Token-Generation Lambda normalizes the claim shape, and this decision supplies the non-spoofable *source* of `custom:tenantId` / `custom:userRole` / `custom:groups` for human principals — the federated counterpart to the M2M App-Client binding. Downstream it makes the entity-access-control plane (AD-19) role/group requirements satisfiable, and the Gateway Interceptor (AD-41) and per-request ABAC (AD-37) consume the normalized `tenantId` regardless of issuance path. Onboarding dry-run check #7 (PRD-017 §4.9) asserts every referenced `claim` resolves to a configured issuance path before a tenant goes `ACTIVE`, so an IdP/mapping mismatch fails at onboarding rather than at first approval.

**Scope note (AD-108).** The demo SPA's interactive login (AD-108) deliberately uses the shared-native-pool + administratively-provisioned-`custom:tenantId` shape rejected above, because per-tenant federation is over-built for a single demo tenant with no external IdP. This does not weaken this decision: production human tenancy remains the per-tenant federated IdP, and AD-108 keeps AD-6's rule that the browser can never self-assign its tenant. The two coexist because they serve different scopes — federated binding for real tenants, native-pool fallback for the demo.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
