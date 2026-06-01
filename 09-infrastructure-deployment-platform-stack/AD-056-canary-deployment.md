# AD-056 — Canary 10% → Per-Quadrant Smoke → 100% with Roll-Forward on Failure

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-56 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-54, AD-5, AD-11

## Context

A bad production deploy should affect as little traffic as possible before it is caught. The system has four distinct workflow paths — one per Kraljic quadrant (AD-5, AD-11) — and a regression on any quadrant would be invisible to a deployment that tests only one path. Blast-radius containment and per-path coverage must both be achieved before full rollout.

## Decision

Deploy to 10% capacity, observe for 10 minutes (error rate, p99 latency, eval quality), run one smoke-test negotiation per Kraljic quadrant, and promote to 100% only on all-pass with no alarms. Any failure rolls forward to the last known-good SHA within 5 minutes (REQ-I103, REQ-I104).

## Alternatives Considered

- **Immediate 100% deploy with post-deploy monitoring.** Rejected: a regression hits all traffic before it is detected; the blast radius is unbounded.
- **Blue/green switch.** Rejected: requires maintaining two full production environments simultaneously; the per-quadrant smoke test is equally achievable with a canary slice at a fraction of the infrastructure cost.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Bounded blast radius — a regression hits approximately 10% of traffic for approximately 10 minutes before rollback is triggered | Deploy speed — a full production rollout takes at least the 10-minute observation window plus smoke tests, rather than flipping immediately |
| Per-quadrant smoke test (AD-5, AD-11) ensures all four routing paths are exercised before full rollout | The 10-minute window still exposes a fraction of traffic to a broken deploy; zero-exposure deploys would require shadow testing infrastructure not currently in scope |

## Results

Satisfies REQ-I103 (canary + per-quadrant smoke) and REQ-I104 (roll-forward within 5 minutes). Rollback reuses AD-54's already-built images and the normal deploy path, so recovery does not introduce new variables. The canary settings (`canary_traffic_pct = 10`, `canary_observation_minutes = 10`) are per-environment overrides in the production tier defined by AD-55.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
