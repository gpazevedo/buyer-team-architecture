# AD-122 — Demo Harness and Test-Tenant App Extracted to Their Own Repository

**Theme:** Infrastructure, Deployment & Platform Stack
**Catalog:** AD-122 · **Source PRD:** PRD-013 · **Status:** Accepted · **Related:** AD-51, AD-75, AD-77, AD-90, AD-101, AD-108

## Context

`demo-harness-project` (demo orchestration and visualization) and `test_tenant_app` (the FastAPI app simulating an ERP-side PR-creation UI, plus its Cognito Hosted-UI login, AD-108) evolved inside the platform repository (`buyer-team-impl`) alongside the 7 production agents, the orchestrator, and all Terraform. Both are demo/test-harness surfaces — neither is code any production tenant runs — but until now they shared the platform's CI pipeline, dependency graph, and release cadence. Widening pyright and pytest to run across the whole repository (same-day change) surfaced concrete friction: the widened check produced a real missing-import error against `demo-harness-project`, which does not cleanly share the main workspace's dependency and type-checking context, and had to be excluded from the check as an interim fix. More broadly, a UI change to the demo harness has no business blocking or slowing a deploy to the production agent fleet, and vice versa.

## Decision

Extract `demo-harness-project` and `test_tenant_app` into a new, independent repository (`buyer-team-demo`) via `git subtree split`, preserving full commit history rather than starting a fresh one. The new repository has its own `pyproject.toml` workspace, its own `.pre-commit-config.yaml` and CI (`ci.yml`), and evolves on its own release cadence. The test-tenant *data* platform — the public-dataset loaders, the `dynamodb-master-data` and `tenant-mdm-emulator` MCP servers, and the Test Tenant Skill (AD-75, AD-77) — is unaffected and stays in the platform repository, since production tenants exercise the same code paths; only the demo-facing consumer app and harness moved.

## Alternatives Considered

- **Status quo — keep everything in one repository.** Rejected: demo/test-harness changes and platform changes shared one CI pipeline and one dependency/type-checking context despite having no architectural coupling; the missing-import friction from the same-day pyright widening was a concrete symptom, not a hypothetical one.
- **Discard the demo harness's history and start the new repository fresh.** Rejected: throws away the record of how the demo evolved for no benefit; `git subtree split` preserves it at no extra ongoing cost.
- **Split into two repositories, one per component.** Rejected: `demo-harness-project` and `test_tenant_app` are used together for every demo run and share no independent release need; one combined demo repository matches how they are actually developed and deployed together.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Demo/test-harness CI and dependency resolution are fully isolated from the platform's — a broken demo dependency can no longer block a platform PR, or vice versa | Shared patterns (the `mypy_boto3_dynamodb` narrow-cast workaround, pyright scope conventions) must now be kept in sync by convention across two repositories rather than enforced by one shared config — demo's pyright scope had already drifted narrower than the platform's before this was caught |
| Demo harness can release on its own cadence, independent of platform deploys | Full commit history was preserved via `git subtree split`, but the platform repository's own copy has not yet been removed — the two repositories currently carry parallel, independently-editable copies of the same code until that cutover happens |
| Smaller, more focused CI surface in each repository | Cross-repository changes (a client contract change both sides depend on) now require coordinating two PRs in two repositories instead of one |

## Results

Shipped in PR #183 (`buyer-team-impl`), landing the code at `gpazevedo/buyer-team-demo`. The new repository's own pyright-warning debt (27 warnings) was fixed same-day in its PR #9, using the same `TYPE_CHECKING`-gated `mypy_boto3_dynamodb` cast pattern the platform repository already used — independently re-derived rather than shared, which is the drift risk this decision accepts. AD-108's Results section is corrected to note the split. The platform repository's own `test_tenant_app` directory has not yet been removed as of this writing; full cutover (deleting the platform-repo originals) is a follow-up, not yet part of this decision's Results.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
