# AD-008 — Production Region Constrained to AgentCore Evaluations Availability

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-8 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-30, AD-79, AD-55

## Context

AgentCore Evaluations is a prerequisite for the platform's closed-loop quality control (AD-30, AD-34). As of 2026-05, Evaluations is GA in only 9 of 16 regions: us-east-1, us-east-2, us-west-2, eu-central-1, ap-south-1, ap-southeast-1, ap-southeast-2, ap-northeast-1, and ap-northeast-2. Per-region active-session quotas also differ: 1,000 in us-east-1 and us-west-2 versus 500 elsewhere. This regional constraint is recorded as assumption ASM-001.

## Decision

Production must deploy only to a region where AgentCore Evaluations is GA. us-east-1 (or us-west-2 as secondary) is preferred because both carry the higher 1,000-session quota.

## Alternatives Considered

- **Deploy to any region and provision Evaluations manually.** Rejected: Evaluations is not available in unsupported regions regardless of provisioning method.
- **Operate without AgentCore Evaluations.** Rejected: Evaluations is the primary online quality-measurement layer (AD-32) whose scores drive automated actions (AD-34); removing it eliminates the closed quality loop.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Evaluations is available everywhere the system runs, enabling the closed-loop quality control the platform depends on | Data-residency and regional-compliance flexibility — tenants requiring EU or APAC in-region residency may not be served in all target regions |
| Higher 1,000-session quota in us-east-1/us-west-2 raises the concurrency ceiling that admission control (AD-79) is derived from | Deployment region is pinned to a platform feature's rollout schedule, not business geography |

## Results

All non-local environments use us-east-1 as the recommended default (PRD-007 §8). The chosen region's session quota directly informs the admission-control capacity budget: G ≈ 100 = `floor(1,000 sessions ÷ 8–10 sessions per concurrent negotiation)` (AD-79). The five-environment model (AD-55) applies the regional constraint to every non-local tier. Any future multi-region or data-residency expansion is gated on Evaluations reaching those regions.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
