# AD-132 — Headline KPIs Computed by a Daily Rollup Lambda; Automation Rate Defined as a Per-PR Deviation Flag

**Theme:** Observability & Evaluation
**Catalog:** AD-132 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-29, AD-34, AD-121, AD-126, AD-133

## Context

PRD-004 §2.3.3 names four headline KPIs — cycle time, automation rate, error rate, cost savings — but the platform emitted none of them. What existed were the raw ingredients: `kpi.cycle_time_days` per terminal PR (AD-29's Layer 3), scattered `procurement/resilience` counters, and Lambda/Step Functions infra metrics. Cycle time is a distribution, not a ratio, so it was directly chartable; the other three are *rates*, and a rate has no datapoint anywhere in CloudWatch unless something computes it. Nothing could alarm on "automation rate dropped" because no such series existed.

Two constraints shaped the answer. First, a CloudWatch alarm that evaluates a Metrics Insights expression *directly* is restricted to a 3-hour evaluation window — unusable for a daily business KPI — whereas Metrics Insights used as a `GetMetricData` **input** carries no such limit. Second, the raw signals are already in CloudWatch as Layer 3 EMF; recomputing rates from DynamoDB would duplicate state the metrics store already holds and require a scan per window.

The first formulation of automation rate then failed on its own terms. It was `1 - (deviated events / count of terminal negotiations)`, summing six event counters (`approval.total`, `kraljic_low_confidence_escalation_count`, `fallback_rejection_count`, `fallback_classification_count`, `suspicious_bid_overflow_count`, `negotiation_timeout_escalation`) over a denominator of *negotiations*. A single PR can emit several of those events, so the ratio was never bounded by [0, 1] — live over 2026-07-19..08-01 it computed to `1 - 8/6 = -0.33`. Four of the six counters also carried no `tenant_id` dimension at all (they are emitted deep in per-node fallback paths that never threaded it through), so every per-tenant figure silently undercounted deviation.

## Decision

Compute the ratio/rate KPIs in a scheduled Lambda and republish them as ordinary metrics. `lambdas/kpi_rollup/handler.py` runs daily on EventBridge (plus on-demand invoke), reads the raw signals over a rolling window through `GetMetricData` Metrics Insights queries, and publishes `kpi.automation_rate`, `kpi.error_rate`, `kpi.pipeline_error_rate`, and `kpi.cost_savings_rate.daily_avg` to `procurement/kpi` — dimensioned `scope=all_tenants` plus one series per tenant — so plain fixed-threshold alarms can watch them. Structurally it is a clone of AD-126's `finops_cost_poller`: same archive/IAM/schedule/on-demand-invoke shape, same batched `put_metric_data` republish. Rollup-as-a-Lambda is the general pattern for any KPI that is a ratio rather than a raw observation.

Define automation rate as `1 - AVG(kpi.deviated)`, where `kpi.deviated` is a **single 0/1 flag emitted once per terminal PR** by `graph_common.emit_pr_cycle_time_kpi`, alongside the cycle-time datapoint it already emitted at every terminal exit. It defaults to `terminal_status != "awarded"` — every cancelled or escalated exit is a deviation by definition — so only the awarded path passes it explicitly, where `node_award_comms` sets `deviated=(approval_decision != "AUTO_APPROVED")`. The average of a 0/1 flag is a rate in [0, 1] by construction, and the flag carries `tenant_id` at every exit, so the per-tenant breakdown is exact rather than approximate.

This narrows the meaning of the KPI deliberately: `fallback_classification_count` no longer feeds it. A degraded-but-successful classification is a **resilience** event, not a human intervention, and it retains its own counter and alarm. Automation rate now means exactly "share of PRs that completed without a human."

## Alternatives Considered

- **Alarm directly on a Metrics Insights expression, no rollup Lambda.** Rejected: alarms evaluating MI directly are capped at a 3-hour evaluation window, which cannot express a daily business KPI. The same expression is unrestricted as a `GetMetricData` input, which is what the Lambda uses.
- **Keep the event-counter formula and just add `tenant_id` to the four un-dimensioned emit sites.** Rejected: this was the follow-up the original implementation named, and it fixes only the attribution half. The numerator would still be a count of *events* over a denominator of *negotiations*, so the ratio stays unbounded and can still go negative.
- **Compute the KPIs from DynamoDB instead of CloudWatch.** Rejected: requires a per-window scan across negotiations and duplicates state CloudWatch already holds as Layer 3 EMF; the raw signals are already metrics.
- **Leave the headline KPIs as raw datapoints on a dashboard, no derived series.** Rejected: a human can eyeball a ratio off two charts, but nothing can alarm on it — the exact gap PRD-004 §2.3.3 exists to close.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The rate is bounded by construction — `AVG` of a 0/1 flag cannot leave [0, 1] — so the class of bug that produced `-0.33` is structurally impossible, not merely fixed | Every terminal exit now emits two datapoints instead of one; the flag is a new write on the hot terminal path |
| Per-tenant attribution is exact, because the flag carries `tenant_id` at every exit rather than at two of six emit sites | Automation rate before and after this change is not comparable — the definition changed, so the series has a discontinuity, and dropping `fallback_classification_count` narrows what counts as deviation |
| Ratio KPIs become ordinary CloudWatch series that any fixed-threshold alarm can watch, free of the 3-hour MI-alarm window | The KPIs are derived and refreshed once daily — a same-day regression is invisible until the next rollup |
| One unambiguous definition of "automation": completed without a human | A definition living in two places — the flag's default in `graph_common`, the awarded-path override in `node_award_comms` — that must stay consistent as new terminal paths are added |

The Metrics Insights dialect is far narrower than SQL, and the constraints are load-bearing enough to record here rather than leave as code comments. Only **one** MI query is permitted per `GetMetricData` call (`ValidationError: Maximum number of queries (1) exceeded`), so the Lambda issues one call per query and concatenates the responses. `GROUP BY` results share a single query `Id` and are distinguished by `Label`, but **the label must be left unset** — CloudWatch then fills in the group's value, whereas an explicit dynamic `${PROP('Dim.<col>')}` label returns `--` for every series, silently collapsing every tenant into one bogus bucket (that syntax is resolved for dashboard widgets, not by this API). There is no `IN`, no `LIKE`, and no `OR` — only `=` and `!=` — and a dimension named after a reserved keyword (`action`) must be double-quoted. Two query rewrites follow directly: `approval_status != 'auto'` replaces `IN ('approved', 'rejected')` (equivalent, because the approval gate emits exactly three values), and node-Lambda selection moved out of the query entirely into a `GROUP BY FunctionName` plus a label-side substring filter, since no wildcard operator exists.

## Results

Realized in `lambdas/kpi_rollup/handler.py` and `infra/modules/observability/kpi_rollup.tf` (Lambda, IAM role, daily EventBridge rule), with four fixed-threshold alarms in `infra/modules/observability/kpi_rollup_alarms.tf`. The `kpi.deviated` emit lives in `orchestrator/graph_common.py`'s `emit_pr_cycle_time_kpi`, with the awarded-path override in `orchestrator/node_award_comms.py`.

**The MI behavior above was verified against live dev CloudWatch on 2026-08-01, after this Lambda failed 44 of 44 scheduled invocations over a month** — every run died on the one-query-per-call limit, so nothing was ever published to `procurement/kpi` by the rollup (the live series in that namespace came from the per-negotiation emitters, carrying `tenant_id`/`quadrant` and never `scope=all_tenants`). The repair and the reformulation shipped together in impl PR #251.

`_present` distinguishes "no PR reached a terminal state in this window" from a legitimate 0.0 deviation rate: a scope is omitted from the output entirely rather than published as a misleading zero.

**Deployment status as of 2026-08-01:** PR #251 is merged but not yet applied — no deploy followed it. Three of the four KPI series (`kpi.error_rate`, `kpi.pipeline_error_rate`, `kpi.cost_savings_rate.daily_avg`) start flowing on the next Lambda deploy; `kpi.automation_rate` additionally requires the orchestrator to ship the `kpi.deviated` emit that feeds it. The four alarms in `kpi_rollup_alarms.tf` were deliberately **kept** through AD-133's alarm cost trim on that basis — they are the one set of currently-unmatched alarms with a known, merged path to matching, unlike the ten AD-133 deleted.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
