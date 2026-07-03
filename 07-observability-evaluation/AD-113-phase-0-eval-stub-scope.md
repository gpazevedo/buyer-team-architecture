# AD-113 — Phase 0 Eval Scope: Ship What's Buildable, Stub the Judge, Gate the Rest

**Theme:** Observability & Evaluation
**Catalog:** AD-113 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-32, AD-33, AD-34, AD-35

## Context

PRD-004's impl doc describes a full evaluator subsystem — LLM-as-Judge and Lambda evaluators wired to AgentCore Evaluations, a Bid Ranking harness, CloudWatch-alarm-driven automated actions (AD-32–AD-35) — none of which had actually been built against `impl/`. Closing the whole gap in one PR was not tractable; some of it (REQ-O219 golden dataset, REQ-O222 tagging, REQ-O223 re-baselining trigger, REQ-O224 CI-gate honesty) was directly buildable against what exists today, while the judge-consistency/calibration requirements (REQ-O220/O221) depend on an LLM-as-Judge evaluator that doesn't exist yet, and the full AgentCore Evaluations wiring is a larger, separate effort. Shipping nothing for O220/O221 would leave the judge-consistency requirement completely unaddressed; building the full AgentCore Evaluations subsystem to satisfy it was out of scope for this PR.

## Decision

Split PRD-004 into what's buildable now versus what's deferred, and make the split legible in code rather than silent:

- Ship the buildable gaps: `kraljic_calibration.json` golden dataset (20→200 rows), `evals/dataset_coverage.py` CI validator, `evals/tagging.py` result-version tagging, `orchestrator/eval_rebaseline_metrics.py` re-baselining countdown, and an honest `run_all.py` docstring stating exactly which evaluator types are CI-gated today (Kraljic only).
- For REQ-O220/O221 (judge consistency/calibration), ship a minimal standalone stub (Phase 0, "option c"): `evals/llm_judge.py` calls Bedrock directly with the Award Rationale Defensibility rubric; `evals/judge_consistency.py` and `evals/judge_calibration.py` run the stddev/MAD checks against it. This is explicitly not the AgentCore Evaluations subsystem described in AD-32 — it is offline, not deployed, and not CI-gated.
- Everything else described in AD-32 through AD-35 (Ground Truth/Lambda evaluator types beyond Kraljic, the CloudWatch-alarm closed loop, ATLAS-specific scheduled evaluators) is explicitly gated behind a later, separate decision rather than silently left as an implied-done design.

## Alternatives Considered

- **Build the full AgentCore Evaluations subsystem now.** Rejected: far larger scope than the concrete gaps (REQ-O219/222/223/224) that were tractable in this PR; would have delayed closing the golden-dataset and CI-gate-honesty gaps, which were the most immediately load-bearing.
- **Ship nothing for REQ-O220/O221 until the full subsystem is built.** Rejected: leaves judge-consistency completely unaddressed indefinitely with no interim signal, and the offline stub is cheap relative to the value of having any judge-consistency check at all.
- **Leave `run_all.py`'s docstring as-is (implying full coverage).** Rejected: this is exactly the kind of silent aspirational-vs-actual gap this decision exists to close; REQ-O224 makes the CI-gate scope explicit in the one place engineers look before trusting the gate.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The concrete, load-bearing gaps (golden dataset size, CI honesty, re-baselining trigger, result tagging) close now instead of waiting on the full subsystem | The AD-32–AD-35 closed loop remains substantially unimplemented; those ADRs' Decision sections are target design, not current behavior |
| A cheap, real judge-consistency/calibration signal exists (offline) rather than none | The stub is standalone and offline — not integrated into AgentCore Evaluations, not deployed, not CI-gated, so it doesn't yet feed AD-34's automated actions |
| `run_all.py`'s docstring makes the CI-gate scope explicit, preventing engineers from trusting a gate that doesn't exist | Anyone reading AD-32–AD-35 without this ADR would reasonably believe the full evaluator set is live |

## Results

REQ-O219/O222/O223/O224 are shipped and, where applicable (O219), CI-gated. REQ-O220/O221 are addressed by an offline stub, not the production subsystem. AD-32 through AD-35's Results sections have been updated to point here and state their actual (mostly unimplemented) status as of this PR. The remaining scope — Ground Truth/Lambda evaluators beyond Kraljic, the CloudWatch-alarm closed loop, and the ATLAS-specific scheduled evaluators — is open and requires a follow-on decision before being picked up.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
