# AD-145 — `lambda_core` Is the Lambda Fleet's Shared Platform Seam

**Theme:** Infrastructure, Deployment & Platform Stack
**Catalog:** AD-145 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-135, AD-101, AD-045, AD-144

## Context

An investigation into whether the 13 Lambdas under `lambdas/` had a common base for cross-cutting concerns found the same drift AD-135 had already fixed for the 6 LLM agents, unfixed and one layer down: 6-plus independent `boto3.client(...)` construction sites with no shared retry `Config`, 7 copy-pasted `logging.getLogger(...)` + `setLevel(INFO)` blocks, and 5-plus hand-rolled `put_metric_data` call sites. Worse than the agent-layer case — where every agent at least had *some* error handling — two handlers, `ad034_halt_negotiations_remediation` and `kpi_rollup`, had **zero exception handling at all**, with no DLQ or on-failure destination configured anywhere in the repo's Terraform for any of the 13. An unhandled failure in either simply vanished after Lambda's default two async-invoke retries, with nothing to alarm on and nowhere for the failed payload to land. Unlike `buyer_agent_core`, there was no pre-existing partial shared package to retrofit from (AD-101's role for agents) — `lambda_core` is built and its consumers migrated in the same effort, rather than in the two stages AD-101 → AD-135 took for agents.

## Decision

A new shared package, `packages/lambda_core`, distributed as a real Lambda Layer via `scripts/build_lambda_core_layer.sh` and `infra/lambda_core_layer.tf` (`aws_lambda_layer_version`, zero bundled third-party dependencies since `boto3` is runtime-provided) — not a zipped dependency duplicated into each handler's own deployment package. Attached to all 13 handlers: root-level Lambdas reference the layer ARN directly, module-owned ones (`master-data`, `observability`, `security`) get it threaded through a `lambda_core_layer_arn` variable. Four modules:

- **`aws.py`** — `get_logger`, and cached `client()` / `ddb_table()` sharing one retry `Config`, mirroring `buyer_agent_core/aws.py`'s shape one layer down.
- **`logging_setup.py`** — the shared `getLogger` + level configuration that replaced 7 copy-pasted blocks.
- **`metrics.py`** — `put_metric` / `emit_failure_metric` (namespace defaults to `procurement/resilience`, best-effort: a broken metrics call must never mask the exception it is trying to report on).
- **`errors.py`** — two failure primitives shaped for the two invocation styles present in the fleet. `guarded` is a context manager (log + metric + **re-raise**) for the SNS/EventBridge-shaped async-invoke handlers, where re-raising is exactly what keeps Lambda's built-in retry and an on-failure DLQ destination firing. `http_error_response` is a decorator (log + metric + a structured JSON error body) for the one Function-URL-shaped handler, `token_exchange_broker`, where a raised exception would otherwise surface as a raw 502 instead of a typed error.

Migration was shape-specific, not uniform: SNS-push remediations and EventBridge-push/scheduled handlers wrap their whole body in `guarded`. `pr_event_router` (DynamoDB Stream, `batchItemFailures`) keeps its existing per-record isolation and adds `emit_failure_metric` inside the per-record `except`, since `guarded`'s whole-body re-raise would defeat partial-batch-failure reporting. `token_exchange_broker`'s catch-all becomes `http_error_response`. `pretoken_normaliser` and `gateway_interceptor` keep their hand-rolled fail-closed catch-alls as-is, because their failure metric names (`security.pretoken.*`, `security.interceptor.*`) are already alarmed on directly in `bucket_a_alarms.tf` and folding them into the generic primitive would mean renaming a live alarm's metric source.

The same effort closes the DLQ gap for the 8 handlers that had zero on-failure destination: a new `lambda_core_dlq` SQS queue (`modules/messaging/main.tf`, 14-day retention, matching the repo's other DLQs' shape) wired via `aws_lambda_function_event_invoke_config.destination_config.on_failure.destination` on each of the 5 `ad034_*` remediations, `guardduty_finding_metric`, `kpi_rollup`, and `finops_cost_poller` — plus the invoked Lambda's own **execution role** (not the invoking SNS/EventBridge principal) granted `sqs:SendMessage`, a requirement the repo's one existing precedent (`modules/step-functions/main.tf`'s `atlas_evaluator_dlq`) already flagged in its own comment. `pr_event_router` (a stream source, not async-invoke) and `dlq_redrive` (a bespoke consumer built for a different DLQ's message shape) were confirmed already correctly wired or intentionally out of scope, not additional gaps.

## Alternatives Considered

- **Leave it as convention, copied from an existing handler when writing a new one.** Rejected: this is the exact status quo that produced the drift AD-135's Context describes for agents, one layer down — and the two zero-exception-handling handlers prove convention alone had already failed silently for an unknown period before this investigation.
- **Extend `buyer_agent_core` itself to also cover Lambdas.** Rejected: Lambdas and AgentCore-hosted agents have materially different runtime contracts — a Lambda has no `/ping` or SIGTERM-drain lifecycle (AD-123), and is cold-start-per-invocation rather than a long-lived session process. Folding both into one package would make AD-135's import-boundary guard ambiguous about which surface it protects, and would tie an agent-layer release to every Lambda handler's dependency footprint.
- **Bundle `lambda_core` as a zipped dependency inside each handler's own deployment package instead of a Lambda Layer.** Rejected: 13 handlers each carrying their own copy reintroduces the drift problem this decision exists to close — a fix would require 13 redeploys instead of one layer publish — and inflates every handler's package with code that never changes per-handler.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One place to tune retry `Config` / logging / metrics-namespace for the whole Lambda fleet — mirrors AD-135's gain for agents, one layer down | An indirection: reading a handler no longer shows its retry `Config` inline; a reader has to know to look at the shared layer |
| 8 handlers that previously vanished failures silently now log, emit a dimensioned failure metric, and land in a DLQ on exhaustion — a gap AD-135's world never had, since agents don't share Lambda's default-retry-then-drop semantics | 8 handlers each grew a new IAM statement (`sqs:SendMessage` on the execution role) — more IAM surface to audit per handler |
| `guarded`'s re-raise keeps this pattern compatible with AD-144's new scheduled-Lambda `Errors` alarms — a swallowed exception would silently defeat both the alarm and the DLQ in the same move | Unlike AD-135, there is no structural import-boundary test yet — a new handler can still hand-roll its own `boto3.client()` and nothing in CI catches it |

## Results

Shipped in impl PR #321 (`feat(lambda_core): shared Lambda Layer + migrate all 13 handlers' cross-cutting concerns`, 3 commits: package foundation + CI wiring, the 13 handler + infra migrations, a CI test-order fix), merged 2026-08-16 (commit `7224317`) and deployed the same day via CI's `dev-deploy` Terraform apply. 1808 tests pass, ruff and pyright clean, `terraform validate` clean. `packages/lambda_core/tests` was added to `pr-checks.yml`'s unit-suite line and to root `pyproject.toml`'s pyright `extraPaths`.

One CI-specific gotcha worth recording structurally: `http_error_response` captures its CloudWatch client at import time via `functools.cache`, so a test fixture that clears the cache and re-patches a client *after* import does not intercept the decorator's own closed-over reference — the fix reads the closure object off the handler before the fixture runs, rather than patching the cache lookup itself. `guarded` does not share this issue, since it reads the module-level client fresh on every call rather than closing over it once.

This closes the same class of gap AD-135 closed for agents, one layer down — with a real DLQ-wiring fix AD-135's world did not need. Unlike AD-135, it does not yet ship a structural enforcement test; a natural follow-on if the same import drift recurs here the way it did before AD-135 existed.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
