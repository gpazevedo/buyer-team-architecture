# AD-115 — `emit_metric` Meta-Alerting: Non-Recursive Fallback Datapoint on Publish Failure

**Theme:** Observability & Evaluation  **Catalog:** AD-115 · **Source PRD:** PRD-004 · **Status:** Accepted — fallback mechanism superseded, see correction below · **Related:** AD-29, AD-34, AD-120

> **Correction (2026-07-08, PR #164/#166):** the "second `put_metric_data` fallback call on a `put_metric_data` failure" mechanism described below **no longer exists.** PR #164 (orchestrator) and PR #166 (agents) replaced the primary emission path with CloudWatch EMF — a local `print(json.dumps(...))` stdout log line, not a network call — specifically to remove the ~20s worst-case blocking I/O this ADR's own Context section describes. Both `metrics.py` implementations now say so directly in their docstrings: *"the old 'meta-alert a second datapoint when put_metric_data fails' fallback no longer applies (there is no remote call left to fail independently of the Lambda's own logging)."* The **outer guarantee still holds** — `emit_metric` remains best-effort and never raises to its caller — but the specific two-tier fallback-datapoint mechanism, and the "Alternatives Considered" analysis weighing it against retries/recursion, describe machinery that has been removed. See AD-29's update for the corresponding Layer-3 decision. The previously-open item (no alarm on the failure signal) is moot along with it: there is no longer a distinct CloudWatch-native failure signal to alarm on — a local `print` failure is close to indistinguishable from ordinary Lambda/process errors already covered by other alarms.

## Context

Every Layer 3 domain metric (AD-29) is published through a shared `emit_metric(namespace, name, value, dimensions)` helper wrapping `boto3 put_metric_data`. That call can fail — throttling, a `boto3` config error, IAM drift, CloudWatch outage — and the call sites it instruments (steering guards, DLQ redrive, negotiation lifecycle nodes, per-invocation token usage) must never let a metrics failure break the business path they're observing. The naive fix, swallow the exception and log a warning, creates a silent blind spot: if CloudWatch itself is unreachable or misconfigured for an extended period, every domain metric this system reports on goes dark at once, and a log line no one is tailing is the only trace. The system needs to know when its own observability pipe is broken, not just avoid crashing because of it.

## Decision

`emit_metric` is best-effort and never raises to its caller. On a `put_metric_data` failure it logs a warning **and** attempts one additional, non-recursive `put_metric_data` call to a fixed low-cardinality namespace (`procurement/observability`, metric `emit_metric.failures`, dimensioned by the namespace/metric name that failed) as a distinct fallback datapoint — not a retry of the original emission, a *report that the emission failed*. If that second call also fails, only the log line remains; there is no further fallback and no recursive call back into `emit_metric` itself, which would risk an infinite loop under a sustained CloudWatch outage. This is implemented twice, independently, because agent processes cannot import orchestrator code (separate Docker build contexts, AD-101): `orchestrator/resilience/metrics.py` and `packages/buyer_agent_core/buyer_agent_core/metrics.py` are separate files with the same signature and the same two-tier behavior.

## Alternatives Considered

- **Log-only on failure, no fallback datapoint.** Rejected: an extended CloudWatch outage or IAM misconfiguration would silently blind every dashboard and alarm fed by domain metrics, with no CloudWatch-native signal that anything is wrong — only a log line in whichever process happened to hit the failure.
- **Retry the original `put_metric_data` call with backoff before giving up.** Rejected: the call sites are in hot paths (steering guards, per-invocation token logging) that must return promptly; a retry loop reintroduces the latency risk this helper exists to avoid, and `botocore`'s own configured retry policy (`max_attempts=2`, `mode=standard`) already covers transient failures below this layer.
- **Recursive `emit_metric` call to report the failure.** Rejected: a failure mode that recurs (e.g., account-wide CloudWatch throttling) would call `emit_metric` again, which could fail again, looping. The fallback path is deliberately a direct, one-shot `put_metric_data` call, not a call back into the function it's reporting on.
- **Shared single implementation instead of duplicating the module.** Rejected: agents and the orchestrator are separate Docker build contexts (AD-101); there is no shared package importable from both without adding cross-image coupling that AD-101 was written to avoid.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A dead metrics pipe is observable in CloudWatch itself (`procurement/observability` / `emit_metric.failures`), not just in scattered logs | A second CloudWatch call on the failure path — under a full outage, both calls fail and the system is back to log-only, same as the rejected alternative |
| No caller of `emit_metric` can ever be broken by a metrics failure — business logic and metrics emission are fully decoupled | Two independently-maintained copies of the same ~20-line function, one per Docker build context, that must be kept behaviorally identical by convention, not by a shared import |
| No infinite-loop risk under sustained CloudWatch failure — the fallback is exactly one extra call, never recursive | `emit_metric.failures` itself has no alarm wired to it yet — the meta-signal exists but nothing currently pages on it (see AD-29 Results) |

## Results

Both implementations ship as of PR #136 (orchestrator) and PR #140 (agents), unit-tested for the failure path in `orchestrator/tests/test_metrics.py` and `packages/buyer_agent_core/tests/test_agent_metrics.py` respectively — same test shape against each module, confirming the two independent copies stay behaviorally identical. Every domain metric call site (steering guards, `kraljic.classification_source`, per-invocation token usage, DLQ redrive counters) inherits this guarantee for free by going through `emit_metric` rather than calling `put_metric_data` directly. The accepted open item is that `emit_metric.failures` is itself unalarmed — it's a real CloudWatch series today, but nothing currently subscribes to or pages on it, so a broken metrics pipe is visible only to someone who thinks to look at that one namespace.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
