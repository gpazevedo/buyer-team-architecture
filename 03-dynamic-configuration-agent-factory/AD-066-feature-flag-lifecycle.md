# AD-066 — Feature-Flag Lifecycle

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-66 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-63, AD-49, AD-25

## Context

Without lifecycle discipline, feature flags accumulate indefinitely. "Temporary" flags become permanent forks in the codebase with no removal path, each one a dead conditional branch that obstructs readability and carries maintenance cost. Flags also need a controlled promotion path from introduction to full production rollout so that a behavioral change is validated at each environment before the next. Neither concern is addressed by simply adding flags to `{env}-system-config` without a policy.

## Decision

Define a five-phase flag lifecycle: Introduction (flag added to `{env}-system-config`, default `false`, deployed to dev) → Testing (enabled in dev, validated via integration tests) → Staging (enabled in staging, validated via E2E and evaluation suite) → Production (enabled in production with alarm monitoring) → Deprecation (removed from the codebase after 2 cycles of always-on; flag key deleted from `{env}-system-config`). Flags are evaluated at agent instantiation only, consistent with the read-once model (AD-63).

## Alternatives Considered

- **No formal lifecycle; flags are permanent.** Rejected: leads to unbounded flag accumulation and dead code branches that obscure the behavioral surface of every agent.
- **Immediate removal when always-on in production.** Rejected: too aggressive; a flag that has been always-on for only one cycle may still be needed for a quick rollback; the "2 always-on cycles" buffer provides a safe rollback window before removal.
- **LaunchDarkly or equivalent external flag service.** Rejected: adds an external runtime dependency; the existing `{env}-system-config` DynamoDB table (AD-63) is sufficient for the flag storage and read-once evaluation pattern the platform already uses.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Controlled rollout — a flag is validated at each environment before promotion to the next | Process overhead: every flag now carries lifecycle bookkeeping and a phase tracking requirement |
| A removal contract — flags do not live forever; the "2 always-on cycles" rule triggers cleanup before dead code accumulates | Some flags linger longer than strictly necessary before removal, because the two-cycle buffer is mandatory even when confidence is high |
| New flags default to `false`, so an introduction never enables behavior unexpectedly in a higher environment | Flag key deletion from `{env}-system-config` must be coordinated with code removal to avoid a missing-key error if any path still reads the flag |

## Results

Flags are stored in the `features` group of `{env}-system-config` (AD-63) and evaluated at instantiation only; they do not change mid-session. The lifecycle is documented in PRD-010 §4. Security-critical flags (`galileo_protect_enabled`, `kraljic_classification_override_enabled`) carry the additional fail-safe default from AD-49 and are exempt from the standard removal contract — they remain as permanent safety controls. The lifecycle provides the controlled-rollout mechanism that AD-63 does not include natively (AppConfig's gradual-rollout machinery was not replicated when AppConfig was retired).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
