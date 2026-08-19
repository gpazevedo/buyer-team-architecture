# AD-142 — ATLAS Navigator Layer Baseline (REQ-S600, REQ-S610)

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-142 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-137, AD-139, AD-140, AD-141

## Context

PRD-005 §10.7's REQ-S600 has specified an ATLAS Navigator layer file (`security/atlas-layer.json`) since the v1.1.0 ATLAS uplift, mapping the §1.3 threat table's 12 tactics to real controls. The 2026-08-12 reconciliation pass (PRD-005 v1.9.3) found this file had been described as built and maintained across three prior changelog entries (v1.7.0, v1.7.3, v1.7.7) without ever existing — `git log --all -- security/atlas-layer.json` in `impl/` returned no history at any commit, the same fabricated-as-built failure mode independently caught for REQ-S608's `agents/shared/memory_auth.py` citation. REQ-S610 (weekly/monthly adversarial robustness eval, results recorded in this file) was blocked on the same missing file.

Tracing REQ-S610 before writing anything found the evaluation half is real and already built: `evals/adversarial_robustness.py:lambda_handler` replays the 60-prompt ATLAS adversarial corpus against every deployed agent on a schedule and emits `adversarial_robustness_score` to `procurement/security` — it simply never had a file to write into, because that file never existed.

## Decision

**REQ-S600.** Create `security/atlas-layer.json` at `impl/` root (the literal path the requirement specifies) with real, sourced mappings for all 12 §1.3 tactics — each entry's `controls` cite an actual requirement ID, metric, alarm, or code path, never a placeholder. Content is a direct transcription of §1.3's own table plus the new REQ-S605 (AD-139) and REQ-S606 (AD-140) controls added in this same session, cross-referenced to the tactic rows PRD-008's IT-ATLAS02 test case already ties them to (system-config tamper → Persistence — Modify Agent Config for GuardDuty; Impact — ML Integrity Erosion for Automated Reasoning). The file's own header explicitly records this history — "fabricated as built in three prior changelog entries, never actually created before 2026-08-13" — so a future reconciliation pass doesn't have to rediscover it.

**REQ-S610's recording half.** PRD-008 REQ-T036 already specifies the real cadence: this file is "refreshed after every red team exercise," independent of the eval's own weekly-staging/monthly-shadow-prod schedule (PRD-008 §Appendix execution table). Read together, REQ-S610's "results shall be recorded in the file" and REQ-T036's exercise-triggered refresh cadence are not actually asking for the same thing — the eval runs continuously and durably via its existing CloudWatch metric; the git-tracked snapshot updates on the slower, human-reviewed red-team-exercise cadence. No new Lambda-to-git (or Lambda-to-S3-to-file) plumbing was added to make every individual eval run write the file live; the "Impact — ML Integrity Erosion" tactic entry instead documents the `adversarial_robustness_score` metric as where continuous results actually live, and carries `last_updated`/`last_red_team_exercise` fields for the periodic snapshot. See Trade-offs for why this fell short of literal live-recording.

## Alternatives Considered

- **Give the Lambda git-push credentials to commit an updated file after every run.** Rejected: no Lambda in this codebase has git write access to any repo (CI does, via `GITHUB_TOKEN`, e.g. `dev-deploy.yml`'s "Commit map back" step) — building that from a Lambda's execution role would be new, security-sensitive infrastructure (a long-lived or dynamically-minted repo-write credential reachable from a Lambda execution role) disproportionate to a ruleset-authoring/baseline pass.
- **Write each run's result to S3 and have `atlas-layer.json` read from it.** Considered — would need the `adversarial_robustness` Lambda (deployed via `infra/modules/step-functions`) to get IAM write access to a bucket, and that module doesn't currently take an S3 bucket reference as an input (only `dlq_archive_bucket` does, for an unrelated purpose). Deferred as a real, scoped follow-up rather than added under this ADR's already-broad scope.
- **A new GitHub Actions workflow, on the eval's own schedule, that pulls the latest metric datapoint and commits an updated file.** The mechanically correct answer given REQ-T036's exercise cadence doesn't literally match REQ-S610's per-run wording — but a new scheduled workflow is a meaningfully separate piece of CI infrastructure, not part of "create the file" as scoped for this pass.

## Trade-offs

| Gained | Given up |
| --- | --- |
| REQ-S600 is genuinely, verifiably real this time — every entry cites a live metric/alarm/code path, and the file documents its own fabrication history so it can't quietly regress to aspirational again | REQ-S610 stays **Partially implemented**: the eval and its metric are real and live, but the file itself does not update automatically on every run — only the eval's own metric does, and the file is a periodic (exercise-triggered) snapshot |
| No new security-sensitive infrastructure (Lambda git credentials, new bucket IAM grants) added under a ruleset-authoring ADR's scope | A real gap remains open: nothing today actually performs the REQ-T036-specified periodic refresh automatically — it is manual/CI-triggered by convention, not enforced. Follow-up: either the GitHub Actions or S3-relay alternative above, scoped as its own change |

**First refresh 2026-08-19 (impl PR #331) — the manual cadence works, and it caught the failure mode this file's own rules exist to prevent.** REQ-S606's entry still described the pre-PR-#306 state ("policy not yet built/attached… no live LLM call today") four days after the Automated Reasoning check went live on Nodes 5 and 7 (AD-140), and again three days after the guardrail split (impl PR #319). The row was re-verified against the account before rewriting, per this file's "cite a real, live control" rule, and now records the AR-carrying guardrail version, both pinned Lambdas, and the known limitation that two of the six rules stay inconclusive. The trade-off row above is the direct cause: with no automated refresh, an evidence row goes stale silently the moment the control it cites changes, and nothing fails. Manual refresh caught it in six days here; nothing guarantees that interval.
| Cross-references PRD-008's already-existing REQ-T036/IT-ATLAS02 rather than inventing new cadence or scenario language | Readers must consult two PRDs (005 and 008) to understand why REQ-S610's "recorded in the file" doesn't mean "every run writes the file" |

## Results

`security/atlas-layer.json` created (12 tactics, all §1.3-sourced, plus REQ-S605/REQ-S606 additions). No code changes to `evals/adversarial_robustness.py` — its existing `adversarial_robustness_score` metric emission is the real-time record; the file is the periodic snapshot. PRD-005 §10.7's REQ-S600 entry moved to Implemented; REQ-S610 moved to Partially implemented, with the recording-cadence reasoning documented in its Status note (spec v1.9.7).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
