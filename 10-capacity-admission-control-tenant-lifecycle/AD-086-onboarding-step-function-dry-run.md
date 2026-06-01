# AD-086 — Onboarding as a Step Function with Dry-Run Activation and Compensation

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-86 · **Source PRD:** PRD-017 · **Status:** Accepted · **Related:** AD-85, AD-42, AD-79

## Context

A new tenant requires many artifacts scattered across PRDs: Cognito identity (AD-42), Skill Runtime, IAM role, Kafka topics, skill config, concurrency rows, Token Vault credentials, and Plugin manifests. Provisioning these ad hoc is error-prone, and a partial failure leaves orphaned resources with no defined cleanup. Previously, the `Σ reserved ≤ G` invariant (AD-79) had a runtime gap — a tenant added via direct DynamoDB write bypassed the config-write-time check. There was also no defined contract for what "validation (dry-run)" actually checks before a tenant is activated.

## Decision

Implement onboarding as a 9-step Step Function, idempotent per step and restartable from any failed step. Include: pre-flight validation (check `Σ reserved ≤ G` at runtime, closing the previous gap); an asynchronous admin-configuration hold via `WaitForCallback` (up to 30 days); a seven-check dry-run before the `PENDING → ACTIVE` transition; and a compensating rollback that reverses provisioning (Steps 7→2) and deletes the row on terminal failure.

## Alternatives Considered

- **Provisioning script (not a Step Function).** Rejected: a script has no built-in idempotency, no restartability from a specific failed step, and no compensation path — a partial failure requires manual cleanup.
- **Synchronous API without a dry-run.** Rejected: activating a tenant before proving end-to-end connectivity means integration failures surface only when real negotiations run, not at provisioning time.
- **Inline validation only (no separate dry-run).** Rejected: pre-flight validation checks preconditions but cannot prove that JWT round-trip, interceptor injection, plugin auth, admission round-trip, and cost emission all work together — only a dry-run execution proves this.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Seven dry-run checks prove end-to-end connectivity (JWT round-trip, interceptor injection, plugin auth, schema validation, admission round-trip, cost emission, claim well-formedness) before `ACTIVE` | Significant machinery: a Step Function, GitHub Actions bridge for Terraform, onboarding-state table, 30-day async hold, and explicit compensation path |
| Compensation guarantees a failed onboarding leaves no orphaned resources | Onboarding can pause up to 30 days awaiting human admin configuration |
| Pre-flight check #2 closes the `Σ reserved ≤ G` runtime gap that previously allowed a direct DynamoDB write to violate the invariant | ~8–15 min runtime (excluding the admin hold) is heavyweight for a provisioning flow |

Dual-control and MFA are required on the destructive endpoints (rollback and purge). The seven dry-run checks each validate a specific cross-PRD integration and catch the M2M `tenantId` claim-issuance-path failures identified in the auth review.

## Results

The dry-run catches integration failures before activation; `~8–15 min` runtime excluding the admin hold. Compensation reverses Steps 7→2 and deletes the `{env}-tenants` row (no tombstone on failed onboarding — see AD-85). Cognito identity provisioning depends on the V3 Pre-Token-Generation trigger (AD-42). The `Σ reserved ≤ G` pre-flight enforces the invariant defined by AD-79 at the moment of onboarding, not only at config-write time.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
