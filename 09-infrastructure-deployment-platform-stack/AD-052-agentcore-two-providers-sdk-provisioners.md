# AD-052 — AgentCore Provisioned Across Two Terraform Providers with SDK Provisioners for Gaps

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-52 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-51, AD-7

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

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
