# AD-056 — Monitoring-Only Observation Window → Per-Quadrant Smoke → 100% with Roll-Forward on Failure

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-56 · **Source PRD:** PRD-007 · **Status:** Accepted (mechanism revised at PRD-007 v1.0.33 — AgentCore has no fractional traffic routing) · **Related:** AD-54, AD-5, AD-11, AD-53

## Context

A bad production deploy should be caught before it is confirmed as the full rollout. The system has four distinct workflow paths — one per Kraljic quadrant (AD-5, AD-11) — and a regression on any quadrant would be invisible to a deployment that tests only one path. Per-path coverage and a bounded confirmation gate must both be achieved before full rollout is declared. A platform constraint shapes the mechanism: AgentCore Runtimes do **not** support fractional traffic routing — there is no microVM-level traffic split, so a true "send 10% of traffic to the new version" canary is not available.

**Naming:** this mechanism is the *observation window* — distinct from AD-131's *variant rollout* and AD-153's *Synthetics canary* (the three were disambiguated by this naming decision).

## Decision

Because AgentCore offers no fractional traffic routing, the mechanism is a **monitoring-only observation window**, not a traffic slice. All images are promoted to production ECR and deployed to every Runtime at once; full rollout is then *confirmed* — not *begun* — only after an observation window (error rate, p99 latency, eval quality) passes with no alarms and one smoke-test negotiation per Kraljic quadrant succeeds. Any failure rolls forward to the last known-good SHA within 5 minutes (REQ-I103, REQ-I104). The mechanism is realized by the `observation-window` job in the `prod-deploy.yml` workflow (PRD-007 §7.4 / impl §11).

## Alternatives Considered

- **Immediate 100% deploy with post-deploy monitoring, no gate.** Rejected: declares rollout complete with no per-quadrant smoke and no defined observation gate; a regression is detected only by ambient alarms, with no structured confirmation step or bounded roll-forward trigger.
- **True fractional-traffic canary (send N% of traffic to the new version).** Rejected: not available — AgentCore Runtimes do not support fractional traffic routing at the microVM level, so this cannot be built on the current platform (PRD-007 §7.4, AD-53).
- **Blue/green switch.** Rejected: requires maintaining two full production environments simultaneously; the observation window + per-quadrant smoke + roll-forward achieves the confirmation gate at a fraction of the infrastructure cost.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A structured confirmation gate — error-rate/p99/eval-quality observation plus per-quadrant smoke must pass before rollout is declared, with bounded (≤5 min) roll-forward on failure | True blast-radius containment: because there is no traffic split, a regression is live on 100% of Runtimes during the observation window, not on a ~10% slice — detection-and-rollback speed, not exposure fraction, is the safety lever |
| Per-quadrant smoke test (AD-5, AD-11) ensures all four routing paths are exercised before full rollout is confirmed | Deploy speed — confirmation still costs the observation window plus smoke tests rather than flipping immediately |

The safety property this decision delivers is **fast structured detection + roll-forward**, not reduced traffic exposure. Zero-exposure or sliced-exposure deploys would require either platform support for fractional routing (absent, AD-53) or shadow-testing infrastructure not currently in scope.

## Results

Satisfies REQ-I103 (observation window + per-quadrant smoke) and REQ-I104 (roll-forward within 5 minutes). Roll-forward reuses AD-54's already-built images and the normal deploy path, so recovery introduces no new variables. The observation settings (`observation_window_minutes`) are per-environment overrides in the production tier defined by AD-55; the former traffic-percentage slice has no effect on AgentCore and is not a traffic control. The constraint that forces this shape is the same immutable-runtime/managed-platform reality recorded in AD-53.

**Update 2026-08-19 (impl PR #332) — reconciling this gate against the workflow found that neither half could have passed a first production run.**

**The observation window watched a hand-maintained alarm list that had drifted.** `eval_rationale_defensibility` and `eval_kraljic_accuracy` (AD-34's actions 4 and 5) were added to `eval_alarms.tf` on 2026-08-15/16 and never to `prod-deploy.yml`, so the window watched 4 of the 6 provisioned eval alarms. The list is now discovered at run time by prefix (`$ENV-buyer-team-eval-`) and guarded by a fail-closed `MIN_EVAL_ALARMS=6`: an empty or partial result fails the deploy rather than passing vacuously against nothing, which matters because AD-133 records these same alarms being deleted wholesale in a cost trim — a silently shrinking alarm set is a demonstrated failure mode here, not a hypothetical one. Watch set 18 → 20. The prefix is deliberately narrow: the two AgentCore online-eval alarms do not match it and stay out of a blocking gate, being alert-only, `notBreaching`, and fed by zero sampled sessions (AD-32). This is the same drift [AD-144](../07-observability-evaluation/AD-144-scheduled-lambda-errors-alarm-registry.md) solves for scheduled-Lambda alarms with a central map — the generalisable rule is that a gate's watch set must be derived from what Terraform provisions, never restated alongside it.

**The per-quadrant smoke could not pass on an environment it had not been run against before.** Three quadrants seeded their own fixtures; NON_CRITICAL was pinned to a pre-seeded dev item, so the suite was dev-shaped rather than environment-portable. It now seeds like the rest, and a new `smoke_fixture_preflight` checks the tenant row and all four categories' suppliers up front, failing with `SMOKE FIXTURE MISSING … NOT a quadrant regression` and the exact seed command. The distinction is the whole point of the fixture: a smoke failure fires roll-forward, so an unseeded environment would otherwise present as a regression and trigger a rollback of a perfectly good image.

**No auto-seed step was added, deliberately.** `scripts/seed_test_tenant.py` `put_item`s the entire governance and model config groups, including `model/default` and `model/previous` — the rows AD-34's Negotiation-Quality model-rollback action restores from. A provisioning step inside the deploy path that silently disarms a rollback mechanism is worse than the failure it prevents, so production seeding stays one-time and out of band.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
