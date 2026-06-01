# AD-083 — Two Independent Kill-Switches (Concurrency vs Spend)

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-83 · **Source PRD:** PRD-016 · **Status:** Accepted · **Related:** AD-81, AD-82

## Context

Concurrency admission (AD-81) and spend enforcement (AD-82) are distinct controls with different risk profiles. Concurrency enforcement is well-understood; spend enforcement is newer and riskier — a budget misconfiguration could wrongly block a legitimate tenant. It must be possible to run spend enforcement in shadow mode (metrics live, no blocking) while concurrency enforcement remains fully active. A single combined kill-switch would force both controls to share the same rollout risk.

## Decision

Implement two independent flags in system-config: `tenant_admission.enforcement_enabled` (concurrency enforcement, default `true`) and `tenant_budget_enforcement` (spend enforcement, default `false` / shadow mode). In shadow mode the accrued-cost counter is maintained and `tenant.budget_utilization` is emitted, but admission is not blocked on budget. When `enforcement_enabled = false`, every admission request is accepted regardless of slot counts — a true global bypass for incident response.

## Alternatives Considered

- **Single kill-switch for both controls.** Rejected: disabling concurrency enforcement during a spend-enforcement incident (or vice versa) would remove both controls simultaneously, increasing blast radius.
- **No kill-switch (always enforce).** Rejected: spend enforcement is new and budget misconfiguration is a realistic failure mode; without an escape valve, a misconfigured budget could block all tenants.
- **Per-tenant flags only (no global switch).** Rejected: a global incident (e.g., a bug in the admission transaction) requires a system-wide bypass, not per-tenant configuration.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Spend enforcement can be validated in shadow mode before blocking production traffic | Two switches instead of one is more operational state to track and document |
| Concurrency kill-switch provides a true global bypass for incident response without touching spend configuration | `tenant_budget_enforcement` defaults to `false`, meaning the absolute spend cap is not protecting tenants until an operator explicitly enables it |
| Independent rollout risk: a spend enforcement rollback does not affect concurrency admission | |

## Results

Flags stored in `{env}-system-config`. Concurrency enforcement is on by default; spend enforcement ships in shadow. The concurrency switch (`enforcement_enabled = false`) admits every request regardless of slots — operators use it to recover from a broken admission transaction without a code deploy. Shadow mode for spend enforcement means counters and metrics are fully exercised in production before the blocking behavior is enabled.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
