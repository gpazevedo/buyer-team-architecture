# AD-042 — Cognito Pre-Token-Generation V3 Normalizes tenantId

**Theme:** Multi-Tenancy & Isolation  **Catalog:** AD-42 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-37, AD-41

## Context

Tenant identity arrives in different shapes depending on the issuance path: Cognito-native users carry `custom:tenantId` as a user attribute, federated users may use a different attribute path, and M2M `client_credentials` flows produce no user attributes at all. The Gateway Interceptor (AD-41) and the ABAC layer (AD-37) require one consistent top-level `tenantId` claim regardless of issuance path. A V2 Pre-Token-Generation trigger assumption in v1.5.0 left M2M tenant isolation undefined because V2 does not fire for `client_credentials`.

## Decision

A shared Pre-Token-Generation V3 Lambda normalizes tenant identity to a top-level `tenantId` claim for all principal types, sourcing from two trusted bindings only: `custom:tenantId` for human users, and the authenticated per-tenant App Client (resolved via a `by-app-client` GSI) for M2M — never from request-supplied `ClientMetadata`.

## Alternatives Considered

- **Pre-Token-Generation V2 trigger.** Rejected: V2 does not fire for `client_credentials` grant flows, leaving M2M tenant isolation undefined — a spec correction discovered in v1.5.0.
- **Resolve tenant from `ClientMetadata` in the M2M flow.** Rejected: `ClientMetadata` is caller-supplied and not authenticated; using it as the tenant binding would allow any caller to claim any tenant identity.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One claim shape for every issuance path: interceptor, audit, and ABAC code is uniform regardless of how the token was issued | Dependency on the V3 trigger specifically, which requires the Cognito Essentials or Plus feature tier — a cost and platform-tier constraint |
| Non-spoofable M2M tenant binding: the App Client secret is the credential, and the GSI resolves tenant identity from it at token issuance | `client_credentials` issues an access token only, so `tenantId` lands on the ID token only for user and federated grants — callers expecting it on M2M access tokens must use the access-token claims |

## Results

The onboarding dry-run (PRD-017 §4.9, check #1) asserts that an M2M token actually carries a top-level `tenantId` claim, catching regressions at the onboarding stage before a tenant goes live. This decision is a prerequisite for AD-41: the interceptor's correctness depends on every JWT carrying a single, non-spoofable `tenantId` regardless of how it was issued.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
