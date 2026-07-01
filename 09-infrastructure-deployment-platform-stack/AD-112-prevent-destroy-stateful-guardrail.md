# AD-112 — `prevent_destroy` as the Guardrail for Dev's Automated Apply

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-112 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-51, AD-111

## Context

Dev runs `terraform apply` automatically on every merge to `main` (AD-111), with no human in the loop. Without a guardrail, a plan that happens to replace a stateful resource — a DynamoDB table, a durable-data S3 bucket, or the Cognito user pool — would destroy and recreate it unattended, losing data with no operator able to catch it beforehand (REQ-I006). Some protection against unattended destructive applies is required; Buyer Team does not provision a KMS key, so any KMS-based protection mechanism is off the table.

## Decision

Every PRD-007-owned stateful resource — the DynamoDB core tables, the `dlq_archive` and `procurement_data` S3 buckets, and the Cognito user pool (merged into its existing `ignore_changes=[schema]` lifecycle block) — carries `lifecycle { prevent_destroy = true }`. This makes `terraform plan`/`apply` fail outright (`Error: Instance cannot be destroyed`) the moment a plan would destroy one of these resources, with no separate approval step or CI gate required to catch it. Removing protection requires a deliberate prior code change to lift the guard, which is itself auditable via git history.

## Alternatives Considered

- **CI plan-gate (auto-apply blocks until a human reviews the plan for destroys).** Rejected: reintroduces a manual step into dev's auto-deploy pipeline, which exists specifically to avoid requiring an operator per merge; also requires additional plan-parsing tooling to detect destroys reliably.
- **Manual-approval gate on every dev apply.** Rejected: same cost as the plan-gate — removes the "auto on merge" property dev deploy is built around (REQ-I106), for protection that only matters in the rare case a plan actually destroys something stateful.
- **KMS-based protection (e.g., customer-managed key with deletion protection).** Rejected: Buyer Team does not currently provision a KMS key; introducing one solely for this guardrail adds an unrelated piece of infrastructure and cost for a problem `prevent_destroy` already solves natively in Terraform.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Dev's auto-apply stays fully unattended for the common case, with no added latency or manual step per merge | The guard is a hard stop, not a warning — a legitimate future need to actually replace one of these resources requires a separate code change to remove `prevent_destroy` first, then a follow-up apply |
| Protection is enforced by Terraform itself at plan time, verifiable in code review, with no separate tooling to maintain | Only covers resources the guard was explicitly added to; a newly added stateful resource that forgets the lifecycle block is unprotected until someone notices |

## Results

Added to `infra/modules/dynamodb/main.tf` (core tables), the `dlq_archive` and `procurement_data` S3 bucket resources, and the Cognito user pool resource. Also applied to the PRD-015-owned `master-data` module's 5 tables in the real code (documented here since PRD-007 does not own that module per its own §3.1 footnote). Live-verified 2026-07-01: a forced local-only replace on `master_load_locks` correctly failed at plan time with `Error: Instance cannot be destroyed`, and was reverted before push — the guard was never bypassed or applied around.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
