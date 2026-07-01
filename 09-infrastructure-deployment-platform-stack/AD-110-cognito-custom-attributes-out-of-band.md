# AD-110 — Cognito Pool Custom Attributes Are Added Out-of-Band via AddCustomAttributes, Not Terraform

**Theme:** Infrastructure, Deployment & Platform Stack
**Catalog:** AD-110 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-52, AD-51, AD-103, AD-6, AD-41

## Context

Per-user multi-tenancy needs a mutable `custom:tenantId` attribute on the Cognito User Pool `{env}-buyer-team-tenants`: the V3 Pre-Token-Generation Lambda promotes it to the top-level `tenantId` claim for user tokens (PRD-005 §4.1 / REQ-S707), so a user with the attribute set maps to *their own* tenant instead of falling through to the single-tenant M2M `by-app-client` binding (AD-6, AD-41). REQ-I001 requires all infrastructure via Terraform.

As-built on dev (2026-07-01), three premises about managing this attribute in Terraform proved false:

- **AWS forbids modifying an existing pool's schema via `UpdateUserPool`.** Adding a `schema` member to a live pool fails at the API with `cannot modify or remove schema items`. The `hashicorp/aws` provider only takes the additive `AddCustomAttributes` path when the schema diff is add-*only*; the live pool carries ~22 auto-created standard OIDC attributes that never byte-match the sparse config, so the provider falls back to a full `UpdateUserPool` — which AWS rejects.
- **The `terraform plan` output is a false negative.** The plan renders the change as `0 to destroy` / in-place update and reads as safe, but it only fails at the apply's control-plane call. A "verified non-destructive plan" that was never actually applied is indistinguishable from one that cannot be applied.
- **Cognito custom attributes are permanent and prefix-trapped.** An attribute can never be deleted or modified once created, and the provider *prepends* `custom:` to the given `name` (it does not strip). A first attempt wrote `schema { name = "custom:tenantId" }`, which physically created the immutable, dead `custom:custom:tenantId` — unusable and unremovable forever.

Without a decision, the correct posture keeps oscillating: `ignore_changes = [schema]` was present originally (PRD-007-impl v1.0.28), removed to "apply the add" (which broke), and re-added — the next contributor who tries to add an attribute the obvious way re-breaks it.

## Decision

Cognito pool custom attributes are provisioned **out-of-band via the native `AddCustomAttributes` API** (boto3 / AWS CLI), documented as an explicit REQ-I001 exception in the same family as AD-52's SDK provisioners. The `aws_cognito_user_pool` resource carries `lifecycle { ignore_changes = [schema] }`, and the `schema` block declares the attribute (`name = "tenantId"`, given **without** the `custom:` prefix, `mutable = true`) purely to document intent — Terraform never reconciles it. `custom:tenantId` was added to the live pool this way (mutable, min 1 / max 256); post-change targeted `terraform plan` is clean.

## Alternatives Considered

- **Add the attribute via `terraform apply` (the schema block, no `ignore_changes`).** Rejected: AWS rejects the resulting `UpdateUserPool` with `cannot modify or remove schema items`; the `0 to destroy` plan is a mirage that only fails at apply.
- **Destroy and recreate the pool with the attribute in its initial schema.** Rejected: destroys all users, app clients, and the per-tenant M2M bindings; unacceptable even in dev and impossible in prod.
- **Rely on the plan-time signal to catch schema drift.** Rejected: the plan reports the doomed change as a safe in-place update, so it provides no protection — the same class of false-safe signal AD-103 rejected for runtime protocol.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The attribute can be added to a live pool at all — the only additive path AWS offers — without destroying users or bindings | IaC purity: attribute creation is an imperative escape hatch Terraform does not track as state (a named REQ-I001 carve-out, like AD-52) |
| `ignore_changes = [schema]` ends the oscillation — the schema block documents intent and applies never fail on it again | A fresh pool is not fully reproducible from `terraform apply` alone; the `AddCustomAttributes` call must be re-run post-create (captured inline + in the provisioning runbook) |

The custom-attribute permanence cuts both ways: the dead `custom:custom:tenantId` stays on the pool forever, but it is inert and the correctly-named `custom:tenantId` sits beside it.

## Results

Realized in `impl/infra/modules/security/main.tf`: the `tenants` pool carries `lifecycle { ignore_changes = [schema] }`, the `custom:tenantId` schema block documents intent, and an inline note records the `AddCustomAttributes` requirement. Live: `custom:tenantId` (mutable, min 1 / max 256) present on `us-east-1_A3DUq3x1c`; two users with distinct `custom:tenantId` each minted their own top-level `tenantId` claim through the real pre-token trigger (no fall-through to the M2M branch), proving per-user isolation on top of AD-6 / AD-41. `scripts/create_tenant_user.py` sets the attribute per user (`admin_create_user`); the web PKCE public client is deliberately **not** granted `custom:tenantId` write access — a public client must not self-assign its tenant. Shipped in `buyer-team-impl` PR #98. This is the Cognito-schema analogue of AD-52 (SDK provisioner for a provider gap) and AD-103 (out-of-band control-plane path guarded because Terraform does not mediate it); PRD-007-impl §Security HCL is to be reconciled as-built (`mutable = true` + `ignore_changes`) to match.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
