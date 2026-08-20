# AD-052 — AgentCore Provisioned Across Two Terraform Providers with SDK Provisioners for Gaps

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-52 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-51, AD-7, AD-130

## Context

As of 2026-05, no single Terraform provider covers all AgentCore resources. `hashicorp/aws` covers `agent_runtime`, `gateway`, `gateway_target`, `workload_identity`, `oauth2_credential_provider`, and `browser`. `awscc` covers `runtime`, `runtime_endpoint`, `memory`, `gateway`, `browser_custom`, and `code_interpreter_custom`. A further set of resources — Evaluator, OnlineEvaluationConfig, Policy/PolicyEngine, Dataset, Harness, PaymentCredentialProvider, BrowserProfile, and ApiKeyCredentialProvider — have no Terraform resource in either provider. REQ-I001 requires all infrastructure via Terraform, creating a direct conflict with platform reality.

## Decision

Provision AgentCore across both `hashicorp/aws` and `awscc` providers and fill the remaining gaps with SDK provisioners (`null_resource` + AWS CLI in CI/CD), documented as explicit exceptions to REQ-I001 with drift-detection smoke tests for each provisioned resource.

## Alternatives Considered

- **Wait for full Terraform provider coverage before deploying.** Rejected: provider parity is not expected before 2026-05 delivery; deferring deployment is not acceptable.
- **Provision all AgentCore resources via SDK/CLI only (abandon IaC for AgentCore).** Rejected: eliminates plan-time visibility and state tracking for the resources that do have provider coverage, breaking REQ-I001 for the covered set.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Platform is deployable today despite incomplete provider coverage | IaC purity — SDK provisioners are imperative escape hatches that Terraform does not track as state |
| Documented exceptions with drift tests make the debt visible rather than hidden | Split across two providers complicates resource cross-references and provider version pinning |
| Time-boxed to 2026-05 — migration to native resources is the explicit expectation as providers catch up | Each provisioner needs its own drift-detection smoke test to compensate for missing state tracking |

## Results

AgentCore Evaluations (PRD-004) and AgentCore Cedar Policy (PRD-005) are the primary CLI-provisioned exceptions. AgentCore Memory is covered by `awscc`; AgentCore Identity (workload identities) is covered by `hashicorp/aws`. Each provisioner has a drift-detection smoke test. These exceptions are the named REQ-I001 carve-outs; they are explicitly time-boxed and expected to migrate to native Terraform resources as the providers mature. AD-53 enforces a related immutability constraint on runtime protocol configuration.

### Addendum (2026-08-20) — the Cedar Policy carve-out has partially closed

`aws_bedrockagentcore_policy_engine` and `aws_bedrockagentcore_policy` now exist in `hashicorp/aws`
(confirmed against the installed 6.60.0 schema; available since 6.47.0, below this repo's `>=
6.55.0` pin — no version bump needed). `aws_bedrockagentcore_gateway.policy_engine_configuration`
was already modeled on the Gateway resource (`infra/gateway.tf`'s `ignore_changes` comment
predates this discovery); the two-phase ordering `manage_policy_engine.py` needed — the Gateway
must exist before its own Cedar policy can name its ARN — turns out to be an ordinary same-apply
Terraform dependency, not a real two-phase constraint: the policy engine has no dependency on the
gateway, so the gateway's `policy_engine_configuration` can reference the engine before the
gateway exists; only the individual policy's statement text (via `replace(file(...), sentinel,
gateway.gateway_arn)`) depends on the gateway. No cycle.

Spiked on `step_functions_orchestrator` (`infra/mcp_gateways.tf`) — smallest blast radius per
AD-148/AD-150 (no live caller, no alarms wired to it): `terraform validate` passes;
`terraform plan` against the live dev backend (scoped to just the new/changed resources) plans 10
creates + 1 in-place update with no errors, and the sentinel correctly resolves against the real
deployed Gateway ARN. **Not applied.**

The observability provisioner turns out to have never been an AgentCore-specific gap at all:
`aws_cloudwatch_log_delivery_source` / `_destination` / `_delivery` are generic, mature resources
(`>= 6.21.0`), and `aws_xray_trace_segment_destination` is already natively Terraform-managed
elsewhere in this repo (`infra/observability.tf`) — the shim just never got migrated onto them.

One real migration hazard the spike surfaced: the existing dev state's
`policy_engine_configuration.arn` already points at a policy engine the old shim script created
(`dev_sfn_orchestrator_policy-a2kf2nbwng`). Applying the native resources as-is would create a
**second, new** engine (Terraform has no state entry for the shim-created one) and orphan the
original rather than adopt it. A real cutover needs `terraform import` of the existing
`aws_bedrockagentcore_policy_engine` / `aws_bedrockagentcore_policy` (and the equivalent
CloudWatch delivery objects, if already provisioned) before the first apply, gateway by gateway —
not a blind migration.

`dynamodb_master_data` (PRD-005 Cedar carve-out) still has no evaluators-related resource type in
the provider as of this check and remains on the AD-52 CLI-provisioner pattern. AgentCore
Evaluations (PRD-004) was not re-checked in this pass.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
