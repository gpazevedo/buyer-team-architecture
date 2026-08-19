# AD-090 — Strands Evals on Every PR

**Theme:** Test Strategy & Quality Gates  **Catalog:** AD-90 · **Source PRD:** PRD-008 · **Status:** Accepted · **Related:** AD-88, AD-75

## Context

Agent quality — classification accuracy, bid-ranking correctness, governance adherence — can regress from a prompt tweak, a tool schema change, or a config change in ways that unit tests do not detect. The system has six agents, eight steering hooks, and a DynamicAgentFactory whose configuration drives model selection and threshold resolution; any of these surfaces can silently degrade agent output quality. Catching such regressions only at staging (G6/G7) means the causal change is days old, buried in a batch of commits, and expensive to diagnose. The evaluation infrastructure — Strands Evals and AgentCore Evaluations — already exists per AD-32/AD-34; the question is when to run it.

## Decision

Run Strands Evals against a versioned golden dataset on every PR, using SimpleLLM, with a < 2-minute budget and a ≥ 95% accuracy gate (G3) that blocks merge. Golden datasets are version-pinned in Git LFS; CI verifies version compatibility before running evals — a version mismatch fails immediately (REQ-T011, REQ-T013). The full evaluation suite (all evaluator types, DefaultLLM for LLM-as-Judge, all critical eval thresholds per PRD-004 §4.5) runs at staging promotion (G7) and blocks promotion on any failure.

## Alternatives Considered

- **Run evals only at staging promotion.** Rejected: the regression is detected days after the causal commit, making diagnosis expensive; the author no longer has full context of the change.
- **Run evals only nightly.** Rejected: regressions accumulate across multiple merges before detection, making root-cause isolation harder and rollback more disruptive.
- **Use full DefaultLLM suite on every PR.** Rejected: DefaultLLM inference cost and latency would make the PR gate too slow (> 2 min budget) and prohibitively expensive for a high-frequency merge cadence.
- **Status quo / no action.** Rejected: without an automated eval gate, agent-quality regressions are caught only by manual QA or staging E2E, both of which are too slow and too late.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Agent-quality regressions are caught at the PR, before merge, where the change is small and the author has full context | PR latency — real Strands Evals run on every push, adding SimpleLLM inference time to the PR pipeline |
| The ≥ 95% gate (G3) makes quality a hard merge requirement, not an advisory signal | The < 2-min / SimpleLLM budget means the PR gate is a fast screen, not the full evaluation suite; some fidelity is sacrificed for speed |
| Golden datasets extracted from the same three public datasets as the test-tenant data (AD-75) ensures eval and ingestion share a consistent ground truth | Golden datasets must be maintained, version-pinned, and kept compatible — a stale or corrupt dataset fails immediately (REQ-T013), which can block the pipeline unexpectedly |

The fidelity gap between the PR-level screen (SimpleLLM, G3) and the full suite (DefaultLLM LLM-as-Judge, G7) is by design: G3 catches regressions cheaply and early; G7 provides complete coverage before any code reaches staging. The two gates compose rather than overlap.

## Results

G3 blocks PR merge at ≥ 95% accuracy with all critical evals passing; the full evaluation suite gates staging promotion at G7. Realized in the GitHub Actions `PR Checks` workflow (< 5 min total including unit tests, lint, and evals). Golden datasets live in `evals/datasets/` (Git LFS, version-tagged with generation timestamps) and are extracted from the Kraljic Strategy, Procurement KPI, and UCI Online Retail datasets — the same sources used for test-tenant data ingestion (AD-75). The `evals/strands_evals/` directory holds EvalSuite definitions for all six agents, the graph, and governance. This gate is part of the PR layer defined by AD-88 and depends on the SimpleLLM cost tier established by AD-57/AD-58 for cost-efficient CI execution.

**AS-BUILT (2026-07-13): none of the above was ever built this way — repo-wide grep for `strands_evals` and `.gitattributes` (Git LFS) both return zero hits.** There is no Strands `EvalSuite` framework usage, no SimpleLLM-tiered PR gate, no Git-LFS-versioned golden dataset. What actually runs on every PR (`.github/workflows/pr-checks.yml`) is two much simpler jobs: **`strands-evals`** (name inherited from this ADR, but the step is `uv run python -m evals.run_all --ci` — the offline, fixture-based Kraljic rule-baseline + coverage + Bid Ranking + Lambda-evaluator + strategy-agent checks documented in [AD-113](../07-observability-evaluation/AD-113-phase-0-eval-stub-scope.md) and [PRDs/agent-evals-overview.md](../../PRDs/agent-evals-overview.md), not a Strands EvalSuite), and **`judge-smoke`** (added PR #193, 2026-07-10 — 4 fixed Bedrock LLM-as-Judge calls against hand-authored good/bad transcripts, `evals/judge_smoke.py`, needs a scoped live-Bedrock credential). Both are true PR-blocking gates, so the Decision's core intent (catch agent-quality regressions before merge, not just at staging) is satisfied — just by a different, simpler mechanism than described above. G7 (full staging suite) is real too, as `staging-eval-suite.yml` (weekly + `workflow_dispatch`, made blocking by PR #185, 2026-07-09). This ADR's Decision/Results sections describe the original target design; treat them as aspirational, not as-built, until this note says otherwise.

**Update 2026-08-15 (impl PR #298): G7's "blocks promotion" behavior is now enforced by the prod-deploy workflow itself, not just documented.** `prod-deploy.yml` gained a `verify-staging-gate` job that every other job `needs`: a pure GitHub-API read (no AWS creds, no GitHub Environment — `actions: read` on the top-level token) of the most recent *completed* `staging-eval-suite.yml` run; its conclusion must be `success` and its `updated_at` within `MAX_AGE_DAYS=8` (weekly cadence + one cycle of buffer), else the whole promotion run fails closed. One looseness was stated in the workflow header rather than papered over: the gate checked the most recent completed run's freshness/result, **not** that the exact SHA being promoted was the one tested. Same pattern as the rest of this ADR's AS-BUILT note: the Decision's intent (staging evals block promotion) realized by a simpler mechanism than the original design described.

**Correction 2026-08-15 (impl PR #304): the exact-SHA gap above closed the same day it was written down, and not via the follow-up this note originally named.** Wiring `evals.tagging.git_sha()` into `staging_suite.py`'s output turned out to be unnecessary — GitHub Actions already exposes `head_sha` on every workflow-run object via its REST API, no code change to the suite needed. `verify-staging-gate` now fetches `staging-eval-suite.yml`'s completed runs, filters to `head_sha == inputs.image_tag` (the SHA being promoted), and gates on the most recent match's conclusion and age — a green run against unrelated code no longer satisfies the gate. The suite's independent weekly cadence still means most promotions need a manual `gh workflow run staging-eval-suite.yml --ref <sha>` dispatch first; that's now stated as the expected operational step rather than a residual gap.

**Update 2026-08-19 (impl PR #332): G7 was structurally unpassable for the whole time it was enforced.** From PR #298 until today, `verify-staging-gate` blocked every production promotion, and five consecutive weekly `staging-eval-suite.yml` runs (2026-07-13 through 2026-08-10) failed on the identical error — `negotiation_quality: calibration SUSPENDED (mad=0.607 > 0.20)` — measured against `transcript_panel.json`'s synthetic placeholder labels. So the gate this ADR describes as "blocks promotion on any failure" was, in practice, blocking on a fixture defect rather than on quality. `staging_suite.py` now treats SUSPENDED calibration as advisory while the panel declares `synthetic_scores: true`, with judge consistency still blocking unconditionally; see [AD-32](../07-observability-evaluation/AD-032-agentcore-evaluations-primary-three-evaluator-types.md)'s resolution note for the fail-safe design. The pattern is the same one this ADR's AS-BUILT note has recorded twice already: the Decision's intent survives, realized by a narrower mechanism than the text describes.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
