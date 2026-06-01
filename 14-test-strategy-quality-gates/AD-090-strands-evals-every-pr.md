# AD-090 — Strands Evals on Every PR

**Theme:** Test Strategy & Quality Gates  **Catalog:** AD-90 · **Source PRD:** PRD-008 · **Status:** Accepted · **Related:** AD-88, AD-75

## Context

Agent quality — classification accuracy, bid-ranking correctness, governance adherence — can regress from a prompt tweak, a tool schema change, or a config change in ways that unit tests do not detect. The system has seven agents, eight steering hooks, and a DynamicAgentFactory whose configuration drives model selection and threshold resolution; any of these surfaces can silently degrade agent output quality. Catching such regressions only at staging (G6/G7) means the causal change is days old, buried in a batch of commits, and expensive to diagnose. The evaluation infrastructure — Strands Evals, AgentCore Evaluations, and Galileo — already exists per AD-32/AD-34; the question is when to run it.

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

G3 blocks PR merge at ≥ 95% accuracy with all critical evals passing; the full evaluation suite gates staging promotion at G7. Realized in the GitHub Actions `PR Checks` workflow (< 5 min total including unit tests, lint, and evals). Golden datasets live in `evals/datasets/` (Git LFS, version-tagged with generation timestamps) and are extracted from the Kraljic Strategy, Procurement KPI, and UCI Online Retail datasets — the same sources used for test-tenant data ingestion (AD-75). The `evals/strands_evals/` directory holds EvalSuite definitions for all seven agents, the graph, and governance. This gate is part of the PR layer defined by AD-88 and depends on the SimpleLLM cost tier established by AD-57/AD-58 for cost-efficient CI execution.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
