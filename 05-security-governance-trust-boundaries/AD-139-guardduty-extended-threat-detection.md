# AD-139 — Enable GuardDuty Extended Threat Detection (REQ-S605)

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-139 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-136, AD-137

## Context

PRD-005 §10.7's REQ-S605 has specified GuardDuty Extended Threat Detection since the v1.1.0 ATLAS uplift (2026-05-22): `CRITICAL` findings should trigger SNS review. The 2026-08-13 verification pass (`project-security-gap-status-2026-08-13`) confirmed no `aws_guardduty_detector` or related resource existed anywhere in `infra/` — this control was never provisioned in any environment, in the same "specified but never built" category as REQ-S606/S607/S600 from the same ATLAS uplift.

GuardDuty findings publish only to the default EventBridge event bus; there is no native CloudWatch metric to alarm on directly, unlike this repo's other REQ-S6xx alarms (REQ-S601, REQ-S608), which alarm on app-emitted EMF metrics. Wiring GuardDuty into the existing `alerts_critical_topic_arn` pattern therefore needs one new piece this codebase didn't previously have: an AWS-service-event source feeding a CloudWatch metric, rather than an application source.

## Decision

Provision `aws_guardduty_detector` at root (`infra/guardduty.tf`, alongside `bucket_a_alarms.tf` per this repo's convention that cross-cutting/alarm-wiring resources live at repo root). Two `aws_guardduty_detector_feature` protection plans are enabled — `S3_DATA_EVENTS` and `LAMBDA_NETWORK_LOGS` — matching this workload's actual surface (Lambda + DynamoDB + Step Functions, no EKS/EC2/RDS). Extended Threat Detection needs no separate opt-in: GuardDuty correlates findings from whichever protection plans are enabled into a single "attack sequence" finding at its Critical severity band (>= 9.0) once those data sources are on.

To close the metric gap, a small Lambda (`impl/lambdas/guardduty_finding_metric`) is the EventBridge target for a rule filtered to `detail.severity >= 9`; it calls `put_metric_data` for `security.guardduty.critical_finding` in `procurement/security`, the same "AWS-service-event → Lambda → CloudWatch metric" shape `gateway_interceptor` (`infra/gateway.tf`) already uses for app-level security counters, so this doesn't introduce a second metrics pipeline. `aws_cloudwatch_metric_alarm.guardduty_critical_finding` reads that metric via Metrics Insights (`SELECT SUM(...) FROM SCHEMA(...)`, matching `guardrail_triggered_recon`/`memory_write_rejected`'s shape) and routes to `alerts_critical_topic_arn` — the same critical-tier SNS topic `governance_confidentiality_violation` uses, since a correlated GuardDuty attack sequence is a critical-tier event by the same reasoning.

## Alternatives Considered

- **EventBridge rule → SNS directly, no Lambda/metric/alarm.** Simpler, and arguably closer to REQ-S605's literal text ("CRITICAL findings triggering SNS"). Rejected in favor of the metric+alarm shape so GuardDuty findings show up in the same CloudWatch-alarm surface as every other REQ-S6xx control (`bucket_a_alarms.tf`), rather than being the one exception with a bespoke direct-to-SNS path — consistent operator experience over marginally less infrastructure.
- **Enable every GuardDuty protection plan (EKS, RDS, EC2 Malware Protection, Runtime Monitoring).** Rejected: this account runs no EKS/EC2/RDS resources, so those plans would bill for coverage of nothing. Scoped to S3 + Lambda, the surfaces that actually exist.
- **A CloudWatch Logs metric filter instead of a Lambda.** GuardDuty findings don't land in a CloudWatch Logs group by default (only EventBridge), so a metric filter has no log stream to attach to without first routing findings to Logs — equivalent complexity to the Lambda path chosen, without reusing an existing pattern.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Closes REQ-S605, the first AWS-native (non-app-emitted) threat-detection signal in this system's alarm surface — GuardDuty catches account-level and network-level threats no app-level guard can see | A new always-on billable resource (GuardDuty detector + two protection plans) and a new Lambda, versus the zero marginal infrastructure of the rejected direct-to-SNS alternative |
| Reuses the exact metric+alarm shape every other REQ-S6xx control uses — one mental model for "how does a security control get to `alerts_critical_topic_arn`" | GuardDuty's `Critical` severity band (Extended Threat Detection correlation, >= 9.0) is a narrower filter than "any GuardDuty finding"; lower-severity findings (Low/Medium/High) are provisioned but not alarmed on in this pass — a deliberate scope match to REQ-S605's literal "CRITICAL findings" text, not full-severity coverage |

## Results

`infra/guardduty.tf` (detector, two protection-plan features, EventBridge rule + Lambda target + permission, CloudWatch alarm) and `impl/lambdas/guardduty_finding_metric/handler.py` added; `terraform fmt` clean. Not yet applied to any environment — pending a `terraform plan` review against the dev backend and explicit go-ahead, per this project's standing rule on infra changes with cost implications. PRD-005 §10.7's REQ-S605 entry moved to Implemented (spec v1.9.5).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
