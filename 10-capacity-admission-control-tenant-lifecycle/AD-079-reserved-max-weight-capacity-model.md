# AD-079 — Reserved + Max + Weight Capacity Model with Global Invariant

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-79 · **Source PRD:** PRD-016 · **Status:** Accepted · **Related:** AD-80, AD-81, AD-8

## Context

Fair-sharing of the AgentCore session pool requires both a guarantee (a tenant must always be able to run some work) and a ceiling (no tenant monopolizes the system). The limit is physical: the AgentCore account-level active-session quota (~1,000) bounds total concurrency, and each negotiation consumes 8–10 sessions. Without a structural capacity model, one tenant's burst can starve another's guaranteed work, and the global ceiling is an untested assumption rather than a derived value.

## Decision

Give each tenant three values — `reserved_slots` (guaranteed floor), `max_slots` (hard ceiling), `weight` (Phase-2 burst tie-break) — stored in system-config. Enforce the global invariant `Σ reserved_slots ≤ G`, where `G ≈ 100` is computed as `floor(active_session_quota / sessions_per_negotiation)` (≈1000 / 8–10). G is configurable as `global_capacity_G` and is raisable via a Service Quota increase.

## Alternatives Considered

- **Global ceiling only (no per-tenant floor).** Rejected: provides no guarantee that a low-traffic tenant can run at all during another tenant's burst.
- **Fixed equal share per tenant.** Rejected: inflexible — different tenants have different negotiation volumes; a uniform share wastes capacity for small tenants and caps large ones arbitrarily.
- **Status quo / no capacity model.** Rejected: without a floor and ceiling, the first high-volume tenant can consume all AgentCore sessions, blocking every other tenant entirely.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Guaranteed floor per tenant that no other tenant's burst can erode | Static reservations hold capacity even when idle — reserved slots are unavailable to the shared burst pool |
| Hard per-tenant ceiling derived from physical session quota rather than a guessed number | `Σ reserved ≤ G` caps the number of tenants that can be onboarded at a given reservation level (surfaces at onboarding, PRD-017 §4.1) |
| Burst tie-breaking via `weight` deferred cleanly to Phase 2 without changing the core model | Rate-limit values feeding G (25 TPS, 100 TPM) are tested assumptions, not published SLAs (validated by load test LD-01) |

## Results

Defaults: `reserved_slots = 5`, `max_slots = 20`, `weight = 1`. The `Σ reserved ≤ G` invariant is enforced at system-config write time and at onboarding pre-flight (AD-86). Drives the three admission regions in AD-80, which are atomically evaluated in AD-81.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
