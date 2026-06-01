# AD-038 — Defense in Depth, Not Replacement

**Theme:** Multi-Tenancy & Isolation  **Catalog:** AD-38 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-6, AD-37, AD-39, AD-70

## Context

When a stronger control (ABAC, AD-37) is introduced alongside existing ones (partition-key checks, Cedar, SQL predicate rewriting), there is a natural pressure to retire the older controls as redundant. In a multi-tenant SaaS system where cross-tenant data exposure is the highest-severity failure class, removing any layer creates a single point of failure — one bypass in the remaining control becomes a breach.

## Decision

ABAC supplements — never replaces — the application-layer `tenant_id` partition-key check (AD-6), Cedar (AD-39), and Plugin SQL/predicate rewriting (AD-70). The stated invariant: no single failure in any one layer produces cross-tenant exposure.

## Alternatives Considered

- **Single canonical isolation mechanism (ABAC only).** Rejected: a single control creates a single point of failure; one bypass in ABAC would produce direct cross-tenant exposure with no other layer to catch it.
- **Remove partition-key checks once ABAC is in place.** Rejected: ABAC does not cover non-partitioned targets; removing partition-key checks would leave the database layer unprotected for any path that bypasses IAM.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The cross-tenant guarantee survives any single control's failure or bypass; layers validate each other | Simplicity of one canonical isolation mechanism; multiple overlapping checks mean more code to maintain |
| Each layer defends against a different failure class (injection, IAM bypass, Cedar misconfiguration, query manipulation), giving independent coverage | More places where a tenancy-related bug could hide — accepted deliberately because the failure being defended is business-critical |

## Results

Codified as a named principle, cited explicitly in AD-37 (ABAC) and in the integration layer (PRD-011 SQL rewriter, REQ-M511). It is the reason the catalog lists tenancy controls at four distinct layers. The four-layer stack: AD-6 (partition keys) → AD-37 (ABAC) → AD-39 (Cedar) → AD-70 (predicate rewriting). Any future tenancy control must state which layer it occupies and confirm the invariant still holds.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
