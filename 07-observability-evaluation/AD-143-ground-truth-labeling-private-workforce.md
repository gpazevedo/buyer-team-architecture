# AD-143 — Ground Truth Labeling via a Private Workforce for Judge Calibration

**Theme:** Observability & Evaluation
**Catalog:** AD-143 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-32, AD-33, AD-34, AD-35, AD-75, AD-90

## Context

Every LLM judge in the stack — Communication Tone and Negotiation Quality (AD-32), plus the Strategy Quality judge — is consistency-checked but **never calibrated**: no human score exists to calibrate against, which AD-32's own notes name as "the defining property of this gap" for strategy quality and the reason `Builtin.Correctness` stays deferred for Award Comms. `evals/fixtures/transcript_panel.json`, the panel that `staging_suite.py` and the judge-calibration tooling read, carries synthetic placeholder scores with an explicit "NOT real human-reviewed labels" header. PRD-004 §4.3.6 / REQ-O221 requires exactly that missing input: dual-reviewer human labels to calibrate the judges against. Without it, AD-34's closed loop runs on scores nobody can prove mean what the thresholds assume they mean.

Labeling real production transcripts was off the table from the start — supplier-facing negotiation text raises confidentiality questions, and AD-75's public-dataset discipline points the same way. The corpus therefore had to be synthetic-but-realistic, the labelers an in-house private workforce, and the whole pipeline far enough outside the deploy path that a labeling hiccup can never block a normal deploy.

## Decision

Run SageMaker Ground Truth labeling jobs over a deterministic synthetic transcript corpus, scored by a private in-house workforce through a custom task UI, with consensus computed by our own reconciliation script:

1. **Isolated Terraform root** (`infra/ground_truth_labeling/`, own state key) — deliberately outside `infra/`'s main root that `dev-deploy.yml` applies untargeted on every push. A labeling-job or workforce misconfiguration must not be able to block a normal deploy; the root is applied manually with explicit `-backend-config`/`-var` for the S3 manifest/output paths.
2. **Private workforce on a dedicated Cognito pool.** The tenant pool (`infra/modules/security/`, ESSENTIALS tier, `prevent_destroy`) is the wrong fit for a labeler workforce. A separate `{env}-buyer-team-labelers` pool + `labelers` group backs the SageMaker private workforce; labelers are added out-of-band (`admin-create-user` + `admin-add-user-to-group`). The portal client runs `USER_PASSWORD_AUTH` with `generate_secret=false`, SageMaker's private-workforce portal requirement.
3. **The task UI is generated from the judge's exact rubric text.** `scripts/render_ground_truth_template.py` writes `ground_truth_template.liquid.html` from the same rubric wording the LLM judges see, so reviewers score against the same definitions the judges use and calibration compares like-for-like.
4. **Two independent reviewers per transcript** (`number_of_human_workers_per_data_object=2`, matching the fixture's `reviewer_1`/`reviewer_2` schema). Consensus per dimension is the mean of the two, computed by `scripts/reconcile_ground_truth_consensus.py` — **not** SageMaker's annotation-consolidation Lambda, which Terraform supports only as a Lambda ARN and which would have discarded the raw per-reviewer answers the reconcile script needs to flag disagreement. Disagreement > 0.10 on any dimension flags the transcript under `_needs_third_review` for a follow-up run.
5. **The corpus is synthetic, deterministic, and alarm-boundary-aware.** `scripts/generate_transcript_panel_corpus.py` (seed 20260702) covers all three panel `source` types, flags ~1/3 as `panel_subset` to preserve the staging-cadence cost profile, and forces deliberately weak/borderline tone and negotiation-quality examples into that subset so the staging suite keeps exercising alarm-boundary cases. Slider values (0–100 integers) divide by 100 to land on the judges' 0.0–1.0 scale; the negotiation-quality composite reuses `score_negotiation_quality`'s formula (mean of 6 consensus components × 25) so calibration compares like-for-like.
6. **Output overwrites the panel in place.** `evals/fixtures/transcript_panel.json` gains real `reviewer_1`/`reviewer_2`/`consensus` scores, so `staging_suite.py` and `judge_calibration.py` consume human labels with no interface changes.

## Alternatives Considered

- **Label by spreadsheet/form and hand-merge into the fixture.** Rejected: no schema enforcement, no audit trail, and the rubric wording drifts from what the judges actually see — the exact calibration-soundness problem this pipeline exists to prevent.
- **Label real production transcripts.** Rejected: supplier-facing negotiation text raises confidentiality questions; the synthetic corpus sidesteps them entirely, the same discipline as AD-75's public-dataset sourcing.
- **SageMaker's built-in annotation consolidation.** Rejected: Terraform's `annotation_consolidation_config` supports only a consolidation Lambda ARN, and the reconcile script needs the raw per-reviewer answers to compute disagreement flags anyway.
- **Status quo — synthetic placeholders forever.** Rejected: the judges stay permanently uncalibrated, REQ-O221 stays open, and AD-34's thresholds remain assertions rather than measured properties.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Real dual-reviewer human labels close REQ-O221 and unblock judge calibration (and eventually `Builtin.Correctness` on Award Comms, AD-32) | A private workforce to operate: a dedicated Cognito pool, out-of-band user creation, and one-shot labeling-job names (Ground Truth never reuses a job name) |
| Raw per-reviewer answers are preserved, so disagreement triage and third-review passes are possible | Synthetic text calibrates rubric alignment, not the production transcript distribution — calibration is to the rubric, not to real traffic |
| A labeling hiccup cannot block normal deploys (isolated root, manual apply) | The root is invisible to `dev-deploy.yml`'s automation — its apply discipline is entirely human |

## Results

Realized in `infra/ground_truth_labeling/` (workforce, Cognito pool/group/client, IAM, labeling job, human task UI, outputs, variables — impl PR #297, 2026-08-15) and three scripts — `generate_transcript_panel_corpus.py`, `render_ground_truth_template.py`, `reconcile_ground_truth_consensus.py` — each unit-tested. Run order before applying: generate corpus → render template → upload the input manifest to S3 → `terraform init -backend-config` + `plan/apply -var` with the two S3 paths. The labeling job has not been executed as of this writing — the pipeline is built and tested, not yet run against the workforce. Once run, the reconcile step overwrites `evals/fixtures/transcript_panel.json` in place, which flows into `staging_suite.py`'s judge-consistency and calibration runs (AD-90's G7) and removes the "synthetic placeholders" caveat AD-32 has carried since its 2026-08-13 update.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
