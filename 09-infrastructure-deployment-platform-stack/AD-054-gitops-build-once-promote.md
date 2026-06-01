# AD-054 — GitOps: Build-Once-Promote, Rollback as Forward Deploy

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-54 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-56, AD-55

## Context

Deployments must be reproducible and auditable end-to-end (REQ-I100, REQ-I101, REQ-I102). Recovery from a bad production deploy must be fast and must not introduce new variables. A per-environment rebuild approach risks producing different artifacts at each stage, violating the "same bytes everywhere" requirement.

## Decision

Git is the single source of truth. Container images are built once on merge to `main` and promoted unchanged through dev → staging → prod; production releases are tagged with semver. Rollback is implemented as a roll-forward deploy to the last known-good SHA (≤5 min, REQ-I104), using the already-built and already-ECR-retained image rather than a separate rollback artifact.

## Alternatives Considered

- **Per-environment rebuild.** Rejected: rebuilding per environment breaks the "same bytes everywhere" guarantee — the image validated in staging may differ from the one deployed to prod.
- **In-place image mutation (tag overwrite).** Rejected: image tags become ambiguous; traceability to a specific git commit (`deployment.git_sha`) is lost.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The exact bytes validated in staging are what run in production — no rebuild drift | Simplicity of rebuilding per environment; promotion requires an ECR retention policy (last 10 images kept) to ensure the good image is still available |
| Rollback reuses the normal, well-tested deploy path and an already-built image; recovery is fast and predictable | Roll-forward assumes a known-good SHA exists and its image is still retained — hence the lifecycle policy is a hard dependency |

## Results

Every deployment is traceable via `deployment.git_sha` (REQ-I100); images are built once and promoted (REQ-I101); production releases carry semver tags (REQ-I102); roll-forward completes within 5 minutes (REQ-I104). This decision feeds directly into the canary deployment strategy (AD-56), which reuses the same already-built images for both canary and rollback paths. The five-environment model (AD-55) defines where each promotion stage lands.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
