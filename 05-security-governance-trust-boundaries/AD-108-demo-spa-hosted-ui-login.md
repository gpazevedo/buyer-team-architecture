# AD-108 — Demo SPA Interactive Login via Cognito Hosted UI (Native PKCE Public Client)

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-108 · **Source PRD:** PRD-013 · **Status:** Accepted · **Related:** AD-42, AD-94, AD-6, AD-37, AD-41, AD-122

> **Correction (2026-07-09, PR #183):** the Results section below lists the `test_tenant_app` frontend and `run_demo.sh` alongside `infra/` and `scripts/` as if realized in one repository. As of PR #183 (see **AD-122**), `test_tenant_app` and `demo-harness-project` are also actively developed in a separate repository (`buyer-team-demo`), extracted via `git subtree` with full history preserved. `infra/modules/security/main.tf`, `scripts/seed_test_tenant.py`, and `scripts/create_tenant_user.py` remain in this (platform) repository. As of this writing the platform repository's own copy of `test_tenant_app` has not been removed, so the two repositories temporarily carry parallel copies — see AD-122's Results. The authentication decision itself — Hosted-UI PKCE, two-tier non-spoofable tenant binding — is unaffected; this is a repository-location correction only.

## Context

The `test_tenant_app` demo SPA needs to exercise the real authentication path — Cognito → JWT → Gateway → ABAC — not just a local no-auth shortcut, or the demo can never prove end-to-end that a browser user's requests carry a non-spoofable `tenantId`. But the two human/machine tenant-binding paths already decided are either unavailable or over-built for a single demo tenant: the M2M App-Client binding (AD-42) is for `client_credentials`, not an interactive browser user, and the per-tenant federated IdP (AD-94) requires an external SAML/OIDC IdP to exist and be wired at onboarding, which no demo tenant has. A demo run must come up with one command and still authenticate against the live dev user pool.

A latent pool defect blocked this: the custom attribute was declared `schema { name = "custom:tenantId" }`, but the AWS provider **prepends** `custom:`, so Cognito physically stored `custom:custom:tenantId` (`mutable=false`), while the V3 Pre-Token Lambda (AD-42) reads `custom:tenantId`. Every Hosted-UI login therefore fell through to the M2M `by-app-client` branch.

## Decision

The demo SPA authenticates human users through the shared Cognito user pool's **Hosted UI** via a dedicated **Cognito-native PKCE public App Client** (`web`, `authorization_code` + S256), gated behind `VITE_AUTH_MODE` (default `dev` = local no-auth; `live` = real Cognito). Tenant binding is resolved in two tiers, both server-side and non-spoofable:

- **Fallback (demo default):** a Hosted-UI user with no `custom:tenantId` resolves tenant from the M2M-style `by-app-client` GSI via a seeded `client#web` row (AD-42 mechanism), mapping all such users to the single test tenant.
- **Per-user (multi-tenant seam):** `scripts/create_tenant_user.py` provisions a user with `custom:tenantId` via `admin_create_user`; the V3 Lambda already prefers that attribute. The correct attribute is restored by an **additive** `schema { name = "tenantId" mutable = true }` (Cognito stores it as `custom:tenantId`), leaving the immutable double-prefixed attribute in place so the change is a pure add, not a pool replacement.

The public browser client is **deliberately not granted write access to `custom:tenantId`** — a browser must never self-assign its tenant.

## Alternatives Considered

- **Per-tenant federated IdP (AD-94) for the demo tenant.** Rejected: over-built for a single demo tenant and blocked on an external IdP that does not exist; it would foreclose the one-command local demo.
- **Keep `dev` no-auth only, no live path.** Rejected: never exercises the real Cognito→JWT→Gateway→ABAC chain, so the demo cannot prove end-to-end authentication.
- **Let the SPA set `custom:tenantId` / bind tenant from browser-supplied metadata.** Rejected: a browser client that can self-assign its tenant is a cross-tenant identity-forgery surface (AD-6, AD-42).
- **Replace the mis-named pool attribute in place.** Rejected: Cognito can never delete or modify a custom attribute; an in-place change forces a pool replacement (user/client loss). The additive attribute is the only non-destructive correction.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The demo exercises the real auth path: PKCE Hosted UI → V3-normalized `tenantId` JWT → Gateway/ABAC, served by the one claim-normalization path (AD-42) | Fallback-bound demo users all map to a single tenant — this is **not** the non-spoofable per-authentication-path binding AD-94 gives production humans |
| A clean per-user multi-tenancy seam exists (`create_tenant_user.py` + `custom:tenantId`) without adopting federation | The mis-named `custom:custom:tenantId` attribute is permanent (immutable), left as dead schema on the pool |

This decision deliberately adopts the "shared native pool + administratively-provisioned `custom:tenantId`" shape that AD-94 **rejected for production** — accepted here only because the scope is a demo tenant and the browser client cannot self-assign tenant. It does not weaken AD-94: production human tenancy remains the per-tenant federated IdP.

## Results

Realized in `infra/modules/security/main.tf` (native `web` PKCE client + `web_client_id` output + additive `tenantId` attribute), `scripts/seed_test_tenant.py` (the `client#web` fallback row) and `scripts/create_tenant_user.py` (per-user provisioning), and the `test_tenant_app` frontend (`AuthGuard` PKCE redirect + token-exchange wait, Bearer injection) with `run_demo.sh` / `docker-compose` honouring `AUTH_MODE`/`COGNITO_*`. Live-validated on dev (`POST /oauth2/token` 200, access token carries `tenantId`, backend 401/200 without/with token). Consumes AD-42's normalization, diverges deliberately from AD-94, and preserves AD-6's rule that the tenant binding never originates from the browser. Realized alongside PR #93 (login path) and PR #94 (attribute correction + per-user tenancy).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
