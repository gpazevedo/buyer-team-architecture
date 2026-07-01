# AD-103 — Skill-Runtime Updates Are Full-Replace; Protocol & Env Re-Asserted via a Guarded Path

**Theme:** Infrastructure, Deployment & Platform Stack
**Catalog:** AD-103 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-53, AD-52, AD-54, AD-101

## Context

AD-53 asserted three things about an AgentCore Runtime's `server_protocol`: it is **immutable** (changing it requires destroy-and-recreate), and a **Terraform plan-time check** is a sufficient guard. As-built on dev (2026-06-28), all three premises proved false for the single MCP skill runtime.

- `update_agent_runtime` is a **full-replace** control-plane call. It flipped the skill runtime's `serverProtocol` MCP→A2A and back **in place** — same runtime id, no destroy-and-recreate. Protocol is **mutable**.
- Because the call replaces the entire runtime definition, **any field omitted from the call silently resets to its default.** Omitting `protocolConfiguration` resets the protocol to the A2A default; omitting `environmentVariables` drops `ENVIRONMENT` / `STATE_MACHINE_ARN`.
- The failure mode is severe and misleading. The skill serves MCP on `:8000`; once reset to A2A, the platform routes receiving-Gateway invokes to the `:9000` A2A contract where nothing listens → every invoke 424s after the ~130 s microVM provision, presenting as a "cold-start" problem. CloudTrail attributed the regression to an ad-hoc boto3 `update_agent_runtime` — a hand-written call that copied the seven agents' `serverProtocol: A2A` onto the one MCP runtime — **not** the CI pipeline, whose Terraform hardcodes MCP.
- Critically, the skill runtime is updated **out-of-band via boto3**, not via routine `terraform apply`, because a broad/targeted orchestrator apply reverts the node Lambdas (AD-52 documents the same out-of-band pattern for the Cedar Policy Engine). So AD-53's plan-time check never sees these updates — the dangerous path is precisely the one Terraform does not mediate.

Without a decision, AD-53's "immutable, caught at plan time" framing leaves the real foot-gun — full-replace resets of omitted fields on the un-checked boto3 path — completely unguarded, and the next hand-written update silently breaks all gateway PO delivery again.

## Decision

Update the single MCP skill runtime **only through a guarded path that re-asserts the full runtime definition every time** — `serverProtocol=MCP`, the live IAM role, the VPC network config, and the canonical env (`ENVIRONMENT` + `STATE_MACHINE_ARN`). Two sanctioned forms, both of which make a protocol/env reset structurally impossible:

1. **`scripts/update_skill_runtime.py --tag <sha> [--wait]`** — a boto3 helper that **hardcodes `serverProtocol=MCP` (not a parameter)**, re-passes the live role and VPC network config (stripping the immutable `requireServiceS3Endpoint`), and **rebuilds env to the Terraform canonical** (resolving `STATE_MACHINE_ARN` live from the procurement state machine) rather than carrying the live env forward. There is no way to call it and get the wrong protocol or a dropped env var.
2. **Targeted `terraform apply -target=aws_bedrockagentcore_agent_runtime.skill -var 'agent_image_tags={"skill-runtime":"<sha>"}'`** — the TF-authoritative form. It reconciles image + env + protocol **and TF state drift** in one shot, and is `-target`-scoped so it does not revert the node Lambdas.

Hand-writing a bare `update_agent_runtime` call for the skill runtime is prohibited. This **corrects AD-53**: the protocol is mutable in place, and the protective control is the guarded re-assertion path, not a plan-time check (which only guards the Terraform path the skill update does not take).

## Alternatives Considered

- **Keep relying on AD-53's plan-time check.** Rejected: it only guards the Terraform path, but the skill runtime is updated out-of-band via boto3 that the check never sees — and the field is mutable, so "immutable, caught at plan time" is factually wrong.
- **Move the skill runtime fully under routine `terraform apply` and drop the boto3 path.** Rejected for now: a broad/targeted orchestrator apply reverts the node Lambdas. The *targeted* skill apply is adopted as the TF-authoritative option, but the boto3 helper stays as the fallback when a targeted apply isn't possible.
- **Treat the regression as a cold-start tuning problem** (raise client timeouts, keep the runtime warm). Rejected: it masked the real cause for an entire investigation cycle; no timeout covers a runtime that has reset to a protocol nothing serves. The cold-start fallback to in-process boto3 delivery is already correct by design (AD-97) and never lost data.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Protocol/env resets can no longer slip in via a hand-written update — the guard re-asserts the full definition every time, `MCP` is non-overridable, and the targeted apply also closes TF state drift | The skill runtime stays a special-cased deploy path (not routine `terraform apply`), with two sanctioned tools to keep in sync with the TF resource definition |
| The true root cause (full-replace resets every omitted field) is documented, ending the "cold-start 424" misdiagnosis | `update_agent_runtime` full-replace semantics must now be respected by anyone touching *any* runtime, not just the skill — the same foot-gun exists for the seven agents |

The mutability correction generalizes: `update_agent_runtime` is full-replace for every AgentCore Runtime, so re-passing `protocolConfiguration` is the same invariant already recorded for Gateways in AD-52 (`update_gateway` is full-replace) and for the A2A agents in `feedback-agentcore-update-resets-protocol`.

## Results

AD-53 corrected — `serverProtocol` is mutable in place via `update_agent_runtime`; the immutable / destroy-and-recreate framing and the "plan-time check is sufficient" claim no longer hold for the out-of-band update path. Realized in `impl/scripts/update_skill_runtime.py` and the targeted-apply runbook. Live: the skill runtime was restored to **v11 READY**, `serverProtocol=MCP`, env `{ENVIRONMENT, STATE_MACHINE_ARN}` restored; post-apply `terraform plan` reported "No changes" (state drift closed); a one-shot receiving-Gateway `receive` returned `HTTP 200` / `RECEIVED`. Shipped in `buyer-team-impl` PR #73 (the helper script + the `deliver_via_gateway` default flip to `true`, gateway PO delivery now on by default with the in-process boto3 fallback intact). Recorded as-built in PRD-005-impl v1.8.3 and PRD-011-impl v1.1.10.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
