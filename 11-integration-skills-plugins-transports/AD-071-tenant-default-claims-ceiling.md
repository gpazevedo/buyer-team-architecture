# AD-071 — `tenant_default_claims` Are the Ceiling for Per-PR Overrides

**Theme:** Integration (Skills, Plugins & Transports)  **Catalog:** AD-71 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-70, AD-19, AD-38

## Context

Authorization for human actions — reading a PR, resolving REQUIRES_ATTENTION, approving a PO — is expressed as JWT `ClaimRequirement` lists (`pr_read`, `pr_ops`, `po_approve`). Each ingested PurchaseRequisition may carry per-PR overrides of these defaults. Per-PR overrides are useful for tightening access on sensitive PRs, but if a PR ingested from a potentially compromised ERP could *broaden* its own required claims, or introduce claim names outside the tenant's IdP, an attacker could weaken authorization from the ERP side without any change to tenant configuration. The ERP integration boundary (AD-69, AD-70) is a trust boundary — data ingested from external systems must not be able to raise its own privilege ceiling.

## Decision

The tenant's `tenant_default_claims` block is the ceiling for per-PR overrides. A per-PR override must be subset-or-equal to the corresponding tenant default (narrowing only, per the rule in PRD-005 §5.5.3), and may only reference claim names in the `allowed_claim_namespaces` whitelist. Both rules are enforced at master-data ingestion (REQ-M111a / REQ-M111b) as a deterministic code-level check in the Skill Lambda, before any domain write. The tenant's four required fields — `pr_read`, `pr_ops`, `po_approve`, and `allowed_claim_namespaces` — are validated at onboarding by PRD-017 dry-run check #7; a default referencing a claim with no configured issuance path is rejected at onboarding rather than failing silently at evaluation.

## Alternatives Considered

- **Allow per-PR overrides to broaden tenant defaults.** Rejected: an ERP-side compromise could introduce weaker authorization requirements for individual PRs, escalating access without a change to tenant configuration.
- **No per-PR overrides — enforce tenant defaults universally.** Rejected: legitimate use cases exist for tightening access on sensitive PRs (e.g., high-value POs requiring a more privileged approver); the narrowing-only rule preserves this flexibility while closing the escalation vector.
- **Validate overrides at evaluation time (Graph Orchestrator interrupt-resume).** Rejected: validation at ingestion time is cheaper, catches the error at the boundary closest to the external system, and prevents invalid data from persisting to the domain store.

## Trade-offs

| Gained | Given up |
| --- | --- |
| An ERP-side compromise can only tighten authorization, never loosen it — the ceiling is tenant configuration, a higher-privilege operation than ERP data ingestion | Per-PR flexibility to broaden beyond tenant defaults is removed; broadening requires changing the tenant default, not the PR |
| Exotic claim names not in the tenant's `allowed_claim_namespaces` are rejected at ingestion, preventing IdP-mismatch failures at evaluation time | Ingestion validation adds a deterministic check per PR that references claim fields; a misconfigured `allowed_claim_namespaces` at onboarding surfaces as a dry-run failure, not a runtime error |
| Defense-in-depth at the integration boundary — closes a privilege-escalation vector that Cedar (tool-level) and ABAC (resource-level) do not address | Broadening a tenant's claim defaults is a higher-privilege operation (requires tenant config update), which may be operationally slower than a per-PR override |

## Results

Four required fields in `tenant_default_claims`: `pr_read`, `pr_ops`, `po_approve` (non-empty `ClaimRequirement` lists per PRD-005 §5.5.2), and `allowed_claim_namespaces`. The narrowing rule is owned by PRD-005 §5.5.3. Claim requirements are evaluated at the Graph Orchestrator interrupt-resume boundary (AD-19), not in agents. The tenant defaults themselves are subject to the same `allowed_claim_namespaces` whitelist as per-PR overrides. Completes the integration-boundary tenancy controls alongside AD-70 (predicate rewriting) within the defense-in-depth stack (AD-38).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
