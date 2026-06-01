# AD-051 — Modular Terraform with S3 Backend, DynamoDB Lock, and Per-Environment State

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-51 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-52, AD-55

## Context

All infrastructure must be reproducible, auditable, and isolated per environment, with no manual console changes in staging or production (REQ-I001, REQ-I002). A single shared Terraform state would allow a development apply to affect production state and makes concurrent applies unsafe.

## Decision

Use modular Terraform with seven domain modules (security, observability, evaluations, dynamodb, messaging, compute, networking) plus an `iam/` and `policies/` directory. State is stored in S3 with SSE-KMS encryption and DynamoDB locking, with separate state per environment (`dev/`, `staging/`, `prod/` key prefixes). Every resource is tagged with `Environment`, `Project`, `ManagedBy`, and `GitSHA`.

## Alternatives Considered

- **Single shared Terraform state across environments.** Rejected: a dev apply could corrupt staging or prod state; concurrent applies are unsafe without per-env locking.
- **Manual console provisioning for AgentCore resources without Terraform provider coverage.** Rejected: removes auditability and reproducibility; REQ-I001 explicitly forbids manual changes in staging/prod.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Per-environment blast-radius isolation — a dev apply cannot touch prod state | Convenience of a single shared state; per-env state plus module boundaries require more wiring and cross-stack references to keep in sync |
| Concurrent applies are locked per environment via DynamoDB; no conflicting applies | More module I/O contracts to maintain as the module count grows |
| GitSHA tag on every resource provides full deploy traceability | — |

## Results

Module I/O table (PRD-007 §2) defines each module's key inputs and outputs. The `terraform-aws-agentcore` community module (aws-ia, awscc-based) is used for most AgentCore resources. Per-environment state satisfies REQ-I001 and REQ-I002. This setup exposes the AgentCore provider-coverage gap that AD-52 addresses, and the per-environment configuration properties that AD-55 governs.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
