# AD-121 — Meta-Observability Heartbeat Dead-Man's-Switch

**Theme:** Observability & Evaluation
**Catalog:** AD-121 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-115, AD-29, AD-34

## Context

AD-115's original two-tier `put_metric_data` fallback-datapoint mechanism was retired when PR #164/#166 moved `emit_metric()` to CloudWatch EMF (a stdout log line, not a network call) — removing the very failure mode, a `put_metric_data` call failing, that the fallback existed to report on. AD-115's correction called the resulting gap moot, since there was no longer a distinct CloudWatch-native signal to alarm on. That reasoning held only as long as nothing needed to watch the EMF pipeline itself. Every Layer 3 domain metric (AD-29) routes through `emit_metric()`, but a business metric only proves the pipe is alive when there's business traffic to emit about — a broken EMF log-line-to-metric extraction during a quiet period (dev idle time, or a genuine production lull) is indistinguishable from "nothing happened," with zero CloudWatch signal either way. Alarming directly on "no business metric arrived" is unsafe specifically because idle periods are real and not incidents, especially in dev.

## Decision

Add a schedule-triggered Lambda (`orchestrator/resilience/heartbeat.py`) that emits its own `pipeline_heartbeat` datapoint to the `procurement/observability` namespace every 5 minutes via the existing `emit_metric()` helper, independent of any business event. A CloudWatch alarm (`aws_cloudwatch_metric_alarm.pipeline_heartbeat`) watches for 2 consecutive missed periods using `treat_missing_data = "breaching"` — the only alarm in the codebase that intentionally treats missing data as a failure, since "no data" is exactly the condition being watched for. The heartbeat's EventBridge rule and the alarm's notification actions are paired into `release_vpc.sh` / `restore_vpc.sh`: an intentional, cost-saving VPC release disables the alarm's actions and then the rule (in that order, so there's no window where the rule is off but the alarm can still notify), and `restore_vpc.sh` re-enables both, rule first.

## Alternatives Considered

- **Alarm directly on "no business metric received."** Rejected: idle periods with zero negotiations are legitimate, especially in dev — this would page on normal quiet time, not on a real pipeline failure.
- **Resurrect AD-115's `put_metric_data`-failure fallback datapoint.** Rejected: there is no longer a remote call to fail independently of the Lambda's own logging (EMF is a local `print`); the mechanism has nothing left to detect.
- **`treat_missing_data = "notBreaching"`, matching every other alarm in the repo.** Rejected: this alarm's entire purpose is to catch the absence of data; treating missing data as non-breaching would make it permanently silent.
- **Leave the heartbeat schedule running through `release_vpc.sh` releases.** Rejected: the heartbeat itself has no NAT dependency (EMF is a stdout print) and would keep ticking fine, but the alarm would then correctly fire on every routine dev VPC release, turning cost-saving into a guaranteed false page.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A dead EMF pipeline is observable in CloudWatch itself, independent of business traffic — including during legitimate quiet periods | A 5th always-on scheduled Lambda and its own IAM role, alongside the 3 existing ones (`runtime_warmer`, `dlq_redrive`, `recovery`) |
| Detection is decoupled from load — works identically for a busy production tenant or an idle dev sandbox | 10-minute detection floor (2 × 5-min periods) is a first-pass, untuned threshold, matching this repo's other early-stage alarms |
| `release_vpc.sh` / `restore_vpc.sh` stay the single source of truth for what's paused during a cost-saving release, rather than leaving an unmanaged always-on resource outside their scope | The disable-actions-then-rule / enable-rule-then-actions ordering is a manual invariant maintained across two scripts, not enforced by any single mechanism |

## Results

Shipped in PR #181: `orchestrator/resilience/heartbeat.py` (handler), `infra/modules/step-functions/heartbeat.tf` (Lambda, logs-only IAM role, `rate(5 minutes)` EventBridge rule), `infra/heartbeat_alarm.tf` (the alarm, kept at root because only root config can wire `module.messaging.evaluation_alerts_topic_arn` into another module's resources — the same reason `agent_runtime_alarms.tf` lives there). The alarm also deviates from house style by setting `ok_actions`, not just `alarm_actions`, so a pipeline-health outage's end is notified, not just its start. The `platform_dashboard.tf` split (see AD-29's update) plots the same `pipeline_heartbeat` metric alongside pure `AWS/*` infra signals, since "is the metrics pipeline itself alive" is a platform-health question even though the metric is custom-emitted. Closes AD-115's previously-"moot" open item with a differently-shaped mechanism than the one AD-115 originally described.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
