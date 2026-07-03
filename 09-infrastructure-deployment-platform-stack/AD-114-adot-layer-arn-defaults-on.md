# AD-114 — ADOT Layer ARN Defaults On at the Root, Not via Workflow `-var`

**Theme:** Infrastructure, Deployment & Platform Stack
**Catalog:** AD-114 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-31, AD-51

## Context

`adot_python_layer_arn` gates the ADOT Lambda layer, `AWS_LAMBDA_EXEC_WRAPPER`, and X-Ray active tracing for the orchestrator node Lambdas and pr-event-router (AD-31). It defaulted to `""` and was never set by any caller, so no ADOT layer was ever attached — the resilience tracer was a no-op and every `agentcore.invoke` span was silently dropped for an unknown period before this was caught live (`Layers: []`, zero spans over a 40-minute window). The var needed a real value from somewhere. This repo's dev workflow does frequent local `terraform apply -target=...` for individual modules rather than always running a full root apply through CI, which rules out setting the value only in a CI workflow's `-var` flag: any local `-target` apply that didn't also pass the var would silently revert the layer to detached, churning tracing on and off depending on who last applied what.

## Decision

Set `adot_python_layer_arn`'s default directly in the root Terraform variable declaration to the AWS-managed us-east-1 x86_64 ADOT Python layer (`aws-otel-python-amd64-ver-1-32-0`), matching the dev runtime (python3.14/x86_64) and the `opentelemetry-api==1.32.0` pin already used in `build_orchestrator_lambda.sh`. Wire the var explicitly into every module that gates on it — the node-Lambda modules (`main.tf:116`) and, closing a separate gap, the `master-data` module (pr-event-router) which had the gating logic but was never passed the var from root.

## Alternatives Considered

- **Set the var via a CI workflow `-var` flag only.** Rejected: this repo does frequent local `-target` applies outside CI; any such apply omitting the flag would silently detach the layer, making tracing coverage depend on who ran the last apply and how.
- **Leave the default empty and require every caller to set it explicitly.** Rejected: this is the status quo that produced the silent multi-week tracing gap in the first place — an opt-in default fails open (untraced) rather than fail-safe (traced).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Tracing is on by default regardless of apply path (full root apply or local `-target`), closing the class of gap that caused this outage | The ADOT layer ARN is now hardcoded to a specific AWS-managed layer version/region/arch combination in the variable default rather than threaded through as an explicit input |
| One root default lights up all currently-gated modules without per-module coordination | Upgrading the ADOT layer version or moving to a different region/arch requires editing the Terraform default rather than a workflow variable |

## Results

Verified live: node Lambdas show `Layers: [aws-otel-python-...]`, `AWS_LAMBDA_EXEC_WRAPPER` set, tracing `Active`; pr-event-router (previously `Layers: []`, `PassThrough`) now matches. A real STRATEGIC PR→PO run produced an `agentcore.invoke` X-Ray span carrying `procurement.negotiation_id`/`agent.name`/`tenant_id`/`model_id`, closing the gap described in AD-31's Results.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
