# AD-111 — Decoupled Per-Component Dev Deploy via Committed Tag Map

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-111 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-54, AD-51

## Context

Dev has 8 independently deployable components (6 agents + skill-runtime + agent-base). Rebuilding and redeploying all 9 on every merge to `main` wastes CI minutes and, more importantly, couples an unrelated component's redeploy to any single change — an operator fixing one agent has no way to ship or roll back that agent alone without touching the other 7 (REQ-I109, REQ-I110). AD-54 established build-once-promote and roll-forward rollback, but at whole-system granularity; it does not address per-component independence within a single environment's auto-deploy pipeline.

## Decision

A git-committed tag map (`infra/image_tags.auto.tfvars.json`) is the sole source of truth for which image tag each of the 9 dev components runs, auto-loaded by Terraform. The auto `dev-deploy.yml` pipeline diffs each component's source against **its own recorded tag** in the map — not the triggering push's `before..after` commit range — so a build failure leaves that component "stale" and it retries on every subsequent run rather than only the one push that touched it. When `agent-base` is stale, every agent rebuilds from it regardless of its own source diff. A separate manual workflow (`deploy-agent.yml`, `workflow_dispatch`) ships or rolls back exactly one component: rollback re-points the map to a prior tag already in ECR, guarded by an `aws ecr describe-images` existence check before any Terraform state is touched. No deploy path may ever pass `-var agent_image_tags=...` directly — doing so collapses the whole map to that value and silently rolls back every other component.

## Alternatives Considered

- **Rebuild and redeploy all 9 components on every merge.** Rejected: wastes CI time on unchanged components and makes it impossible to ship or roll back one agent without redeploying the rest, worsening blast radius for an unrelated bug.
- **Global `-var agent_image_tags=...` override for one-off ships.** Rejected: collapses the entire tag map to the passed value, silently rolling back every component not named in the override — a single-component intent producing a whole-system side effect.
- **Staleness computed from the triggering push's `before..after` range.** Rejected: not self-healing — a failed build is only retried if a *subsequent* push happens to touch that component's source again; the failure could otherwise go unnoticed indefinitely.

## Trade-offs

| Gained | Given up |
| --- | --- |
| An operator can ship or roll back one component without touching the other 8, and CI doesn't rebuild unchanged components | An extra committed artifact (the tag map) must stay in sync with reality; a hand-edited or merge-conflicted map can desync from what's actually deployed |
| Self-healing staleness (diffed per-component, not per-push) means a failed build is retried automatically on every later run, not just the push that caused it | Slightly harder to reason about "what will this push deploy" — the answer depends on the map's current state, not just the diff of this commit |
| Manual rollback reuses an already-built ECR image and fails fast before touching Terraform state if the tag doesn't exist | `workflow_dispatch` is only invocable once the workflow file exists on the default branch, even when dispatched against another ref — an operational gotcha discovered live |

## Results

Realized in `dev-deploy.yml` (self-healing auto path, direct push `6109678`) and `deploy-agent.yml` (manual ship/rollback, PR #106). Live-verified against real dev AWS 2026-07-01: PR #106 merge triggered a correct full build+apply; a subsequent no-op push correctly skipped all 3 build jobs; a manual `spot-bidding` ship and rollback each completed in under 6 minutes touching only that component's Terraform diff. This refines AD-54's build-once-promote model to per-component granularity within a single environment, and depends on the ECR keep-last-10 retention policy (AD-54) to guarantee a rollback target still exists.

The "extra committed artifact must stay in sync with reality" risk named in the Trade-offs table above materialized in practice: the deploy job's final "commit map back" push (`git push origin HEAD:main`) is a plain fast-forward push, and a long build+apply window gives `main` room to advance underneath it (another merge, another deploy). A rejected push previously just failed silently — the tags Terraform had actually applied never landed in the git-committed map, so the *next* run's self-healing diff started from a stale source of truth. PR #233 (2026-07-20) closed this: `actions/checkout@v7` now uses `fetch-depth: 0` so a rebase has a merge base, and the push step rebases the single map commit onto `origin/main` and retries up to 5 times before failing the job. This doesn't change deploy behavior on the happy path — it makes the one write path to the map resilient to the exact race this ADR's own design invites by keeping deploys decoupled and concurrent.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
