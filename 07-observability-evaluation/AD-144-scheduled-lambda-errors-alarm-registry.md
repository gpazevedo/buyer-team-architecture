# AD-144 — Every Scheduled Lambda Registers for an Errors Alarm via a Central Map

**Theme:** Observability & Evaluation
**Catalog:** AD-144 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-121, AD-132, AD-133, AD-045, AD-034

## Context

Two independent incidents went undetected for as long as they did because a scheduled Lambda's own execution failures had no alarm watching them. `approval_sweep` ran into an AccessDenied error on every invocation for 12 days before anyone noticed. `kpi_rollup` failed 44 out of 44 runs over 30 days while its own business alarms — the ones reading the metrics `kpi_rollup` itself would have emitted on a successful run — stayed green throughout, because `treat_missing_data = "notBreaching"` makes "the Lambda never ran successfully enough to emit anything" indistinguishable from "there was nothing anomalous to report." Nine `schedule_expression` Lambdas existed in `infra/` (`outbox_poller`, `dlq_redrive`, `runtime_warmer`, `heartbeat`, `adversarial_robustness`, `supplier_skew`, `approval_sweep`, `kpi_rollup`, `finops_cost_poller`) and most had no `AWS/Lambda` `Errors` alarm at all. These are fire-and-forget EventBridge-triggered functions with no synchronous caller to propagate a failure back to — nothing upstream of the alarm layer itself would ever notice a scheduled Lambda that throws on every run.

## Decision

Register every `schedule_expression` Lambda's `Errors` metric in one of two `for_each`-driven maps rather than hand-writing a one-off `aws_cloudwatch_metric_alarm` resource per Lambda: `local.scheduled_lambdas` in `infra/modules/step-functions/alarms.tf` for the Lambdas that module owns (`outbox_poller`, `dlq_redrive`, `runtime_warmer`, `heartbeat`, `adversarial_robustness`, `supplier_skew`, `approval_sweep`), and a second map in the new `infra/modules/observability/scheduled_lambda_alarms.tf` for the two the observability module owns (`kpi_rollup`, `finops_cost_poller`). Every alarm in both maps shares one shape: `Namespace = AWS/Lambda`, `MetricName = Errors`, `Period = 300`, `Statistic = Sum`, `Threshold = 1`, `EvaluationPeriods = 1`, `treat_missing_data = "notBreaching"` — deliberately the opposite choice from AD-121's heartbeat, since a scheduled Lambda that simply hasn't run yet (fresh deploy, low-frequency schedule) is not itself a failure the way a missing heartbeat tick is — and every alarm notifies the shared `evaluation-alerts` SNS topic. A `moved` block re-pointed the pre-existing `approval_sweep` alarm into the new map without destroying and recreating it, preserving its alarm history across the migration.

## Alternatives Considered

- **Hand-write a one-off alarm per new scheduled Lambda, added when someone remembers.** Rejected: this is the exact status quo that produced both incidents — `approval_sweep` and `kpi_rollup` are Lambdas that already existed, unalarmed, for long enough to fail silently for 12 and 30 days respectively.
- **Alarm on each Lambda's own business metrics going quiet, mirroring AD-121's heartbeat pattern.** Rejected: `kpi_rollup` already had business alarms and they stayed silent through the entire 30-day incident, because a handler that fails before emitting anything looks identical to a handler with nothing to report. `AWS/Lambda` `Errors` is emitted by the Lambda service itself, independent of what the handler code does or fails to do, making it the one signal that survives a handler failing before it emits anything of its own.
- **A CI/lint check that fails when a new `aws_lambda_function` with `schedule_expression` has no matching alarm entry.** Considered, not built this round: the two central maps are a lighter-weight version of the same guarantee — a Lambda left out of both maps is still silently unmonitored, so this is a convention enforced by review, not yet a structural guarantee the way AD-135's import-boundary test is for agent code. Left as a plausible follow-on if a tenth scheduled Lambda ships without registering.

## Trade-offs

| Gained | Given up |
| --- | --- |
| All 9 scheduled Lambdas now alarm on their own execution failures, closing the exact blind spot both incidents exploited | A scheduled Lambda added to Terraform without also being added to one of the two maps is still silently unmonitored — the map is a convention, not (yet) a CI-enforced guarantee |
| `approval_sweep`'s `moved` block preserved continuous alarm history through the migration — no gap, no destroy/recreate | Two separate maps, split by which Terraform module owns the underlying Lambda — a new scheduled-Lambda author has to know which module they're in to register in the right one |

Dev's alarms were cost-paused (0 live, per AD-133's delete-and-replay pattern) at the moment this merged, so the new alarms take effect only on the next `restore_vpc.sh` replay — they do not protect anything until then.

## Results

Shipped in impl PR #320 (`feat(observability): scheduled-Lambda error alarms + per-tenant cost.usd metric`), merged 2026-08-16: `infra/modules/step-functions/alarms.tf` and the new `infra/modules/observability/scheduled_lambda_alarms.tf`. The same PR's `cost.usd` metric addition is a separate, unrelated change to the same file (`orchestrator/resilience/agent_invocation.py`) and is recorded as an update to AD-061, not here. Closes the observability gap `kpi_rollup`'s own automation-rate deviation flag (AD-132) could not have caught, since that flag depends on the Lambda running at all.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
