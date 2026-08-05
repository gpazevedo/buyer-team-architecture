# AD-133 — Delete and Replay Dev Alarms Across a Pause Window; Suppressing Alarm Actions Saves Nothing

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-133 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-29, AD-30, AD-34, AD-121, AD-126, AD-132, AD-24, AD-43

## Context

The dev environment is paused routinely to save money: `release_vpc.sh` tears down the NAT Gateway, its Elastic IP, and the interface endpoints, and `restore_vpc.sh` puts them back. Over time that script pair accreted the other things worth pausing — AD-30's X-Ray Transaction Search, AD-126's daily FinOps poller rule, AD-121's heartbeat rule — on the principle that the two scripts stay the single source of truth for what is suspended during a cost-saving window.

A cost review on 2026-08-01 found alarms were **87% of the CloudWatch bill** ($4.16 of $4.77 over 30 days), and that a meaningful slice of that bought nothing at all. The premise everything else follows from: **CloudWatch bills $0.10 per alarm-month, prorated hourly, regardless of the alarm's state and regardless of whether its actions are enabled.** AD-121's mechanism — disabling the heartbeat alarm's *actions* across a pause, in a carefully ordered dance with its EventBridge rule — therefore suppressed a false page correctly and saved exactly $0. Only deleting an alarm reduces the bill.

The same review found alarms billing for coverage that did not exist. Fourteen agent-runtime alarms had matched zero series since they were created (AD-29's Results carries that correction). Four evaluation-score alarms (AD-34) were alert-only signals on an SNS topic with no subscriptions, with none of AD-34's five automated responses ever wired. Six per-node Lambda duration alarms duplicated failure signal already carried by `node_lambda_errors` and `sfn_executions_failed`.

## Decision

Delete dev alarms across a pause window rather than muting them, and delete outright those that bill for zero coverage.

`infra/alarm_toggle.py` snapshots every alarm under the `${ENVIRONMENT}-buyer-team-` prefix — full `put_metric_alarm` spec plus tags — to a gitignored file and deletes them. `release_vpc.sh` runs that first, before anything is torn down; `restore_vpc.sh` replays the snapshot **last**, once egress, schedules, and tracing are all back, so nothing fires on state that existed only mid-restore. The prefix scope leaves unrelated alarms in the shared account alone.

Separately and permanently, delete ten alarms that cover nothing: AD-34's 4 evaluation-score alarms and the 6 per-node Lambda duration alarms, taking the dev footprint from 55 to 45. Both deletions are tombstoned as comments in the Terraform they were removed from, so the context stays discoverable at the site rather than only in git history. The 4 KPI alarms (AD-132) are deliberately kept: they are unmatched for a reason with a merged fix, not for lack of coverage.

## Alternatives Considered

- **Keep AD-121's disable-alarm-actions mechanism.** Rejected: it saves $0. Alarms bill by the alarm-month whether or not actions are enabled, so action suppression addresses noise and nothing else — and once alarms are deleted for the window, the noise problem is solved as a side effect.
- **`terraform apply -target` the alarms on restore instead of replaying a snapshot.** Rejected: every alarm references the resource it watches, so targeting the alarms drags their dependencies into the plan — roughly 75 tag rewrites plus a new `agent_runtime_version` cut on all 7 AgentCore runtimes, on every single pause cycle. A snapshot replay touches nothing but the alarms.
- **Leave the alarms running through the pause.** Rejected: they are the largest single line on the CloudWatch bill, and with egress gone and the schedules suspended most of them (errors, latency, backlog, heartbeat) would breach or go to no-data on what is a routine, intentional release.
- **Move alarms out of Terraform entirely so the toggle owns them end to end.** Rejected: that trades a bounded, self-closing drift window for permanent loss of plan-time visibility and drift detection on the steady state, to solve a problem that exists only during a pause.

## Trade-offs

| Gained | Given up |
| --- | --- |
| ~$4/mo of dev alarms stops billing while dev is paused, and $1.00/mo permanently from the ten zero-coverage deletions | Terraform state still lists the alarms while they are deleted — genuine drift, closed only by the replay |
| The replay touches only alarms, avoiding the ~75 tag rewrites and 7 runtime-version cuts a targeted apply would incur each cycle | The snapshot is point-in-time: a Terraform-side alarm change applied mid-pause is replayed as an orphan Terraform no longer tracks, so alarm edits must be applied while dev is **up** |
| Noise suppression comes free — a deleted alarm cannot page on a paused environment, so AD-121's ordering invariant disappears rather than needing maintenance | Alarm state history does not survive the round trip; alarms return in `INSUFFICIENT_DATA` and re-evaluate on the next datapoint |
| The same imperative pause-toggle pattern already used for X-Ray (AD-30) and the FinOps poller (AD-126) — a now-established shape, not a new mechanism | `describe_alarms` returns an empty `Dimensions` alongside `Metrics`, a pair `put_metric_alarm` rejects — so every metric-math and Metrics Insights alarm fails to replay unless the single-metric fields are stripped first |

The Terraform drift is the same AD-52-shaped debt those other toggles already carry, and it is self-limiting in the same way: a broad `terraform apply` during a pause simply re-asserts the alarms, which is harmless because it recreates exactly what the restore would have.

## Results

Realized in `infra/alarm_toggle.py`, wired into `infra/release_vpc.sh` (first step) and `infra/restore_vpc.sh` (last step), with the snapshot path gitignored. The round trip is verified byte-identical including tags. Shipped in impl PR #252, applied while dev was up.

**AD-121's disable-actions/enable-actions ordering invariant is retired by this decision** — see that ADR's 2026-08-01 update. Its heartbeat alarm is now snapshot-deleted and replayed with every other dev alarm; the pairing that remains in those scripts is the EventBridge rule alone.

**AD-34's detection half is now fully gone in dev**: `infra/modules/step-functions/eval_alarms.tf` was deleted, with the rationale tombstoned in the sibling `alarms.tf`. The underlying `procurement/evaluation` metrics still flow and are still charted on the application dashboard — only the alarms are gone. The 6 per-node duration alarms are tombstoned in the same file, flagged for re-add in prod where approaching-the-timeout is worth knowing before the failure lands.

The same PR fixed the 14 dead agent-runtime alarms rather than deleting them (AD-29's Results correction), which is the counterpoint that makes this decision coherent: alarms that cover nothing get deleted only when nothing is going to make them cover something.

**The mid-pause constraint was exercised for the first time in impl PR #253** (2026-08-03), which adds a `governance_hook_error` alarm (AD-24). It was merged to Terraform and deliberately **not applied**: dev was mid-pause with 0 live `dev-buyer-team-*` alarms, and the snapshot waiting to be replayed predates the new alarm, so applying would have created something the restore would then not know about. The trade-off row above — "alarm edits must be applied while dev is **up**" — is the operative rule, and it held. The dev footprint is 45 alarms live, 46 in Terraform until that apply happens.

That PR also inherits this decision's unresolved caveat: the new alarm publishes to `dev-buyer-team-evaluation-alerts`, the same 0-subscriber topic whose emptiness justified deleting AD-34's evaluation alarms here. An alarm on a broken safety guard that pages nobody is, by this ADR's own standard, a deletion candidate rather than coverage — the resolution is a subscription, not an exception.

**Update 2026-08-05 (impl PR #259) — the unresolved caveat above is resolved.** `evaluation_alerts` now has a real subscriber: `module.messaging.aws_sns_topic_subscription.evaluation_alerts_email`, gated on `var.alert_email` (empty default, so this is opt-in per environment). Applied to dev and confirmed live via `sns.list_subscriptions_by_topic` — see [AD-43](../05-security-governance-trust-boundaries/AD-043-bedrock-guardrails-all-agents.md)'s 2026-08-05 update for the targeted-apply details (scoped narrowly around the paused-VPC and stale-`git_sha`-default hazards a full plan would have hit). This ADR's own standard — "the resolution is a subscription, not an exception" — is now met. It is met for the *topic*, not yet for `governance_hook_error` specifically: per [AD-24](../02-agent-architecture-behavioral-control/AD-024-steering-hook-failure-semantics.md)'s Results, that alarm is a separate resource from the subscription and remains merged-but-not-applied to dev, still waiting on the pause-safe apply window this ADR's own trade-off table requires. Separately, the 2026-08-01 update's re-add condition for AD-34's deleted evaluation alarms ("a subscriber and at least one wired action") is now half-satisfied for the first time since this decision — no wired action exists yet.

**A guard failure worth recording.** The new release step shells out through `uv run python`, but `infra/tests/test_vpc_scripts_ordering.py` stubbed only `aws` and `terraform` on `PATH` — so running the test suite reached the real dev account and deleted all 55 alarms for real. They were replayed from the snapshot the same step had just written, with zero config or tag drift, which is itself evidence the round trip works. The test now stubs `uv` as well, asserts it resolves to the stub before running anything, and points `ALARM_SNAPSHOT` at `tmp_path`. The two ordering tests were re-pointed from the retired disable/enable-actions calls to the equivalent property against the new ones — alarms removed before the heartbeat rule is disabled, recreated after it is re-enabled — plus a third asserting release snapshots and deletes in one step, since a delete without a written snapshot leaves nothing to replay.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
