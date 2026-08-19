# AD-147 — Cedar Principal Tags Carry `tenantId`, Unblocking Per-Tenant Policy

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-147 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-38, AD-39, AD-52, AD-127

## Context

Cedar runs on exactly one surface — the receiving Gateway (AD-39), in `LOG_ONLY` — and its
policy is a coarse outer gate: any action on that Gateway for a principal carrying a `receive`
OAuth scope. Every proposal to widen Cedar's footprint (the ingest Gateway, the three ungated
MCP runtimes, per-tool policy) depends on one unanswered question: *what can a policy actually
say about the caller?*

For an M2M `client_credentials` token the Cedar principal is `AgentCore::OAuthUser::"<sub>"`,
and that `sub` is the Cognito App Client id — not a user, not a tenant. Only `scope` was known
to surface as a principal tag. `lambdas/pretoken_normaliser/handler.py` (REQ-S707/S708) already
mints a top-level `tenantId` claim on every token, but whether AgentCore projects that claim
onto the Cedar principal the way it projects `scope` was never verified. `infra/token_exchange_broker.tf`
blocks its own cutover on exactly this question, and without an answer every per-tenant Cedar
predicate is speculative — policy that may be unable to express what it needs to express.

## Decision

**`tenantId` resolves as a Cedar principal tag, with the correct per-tenant value.** Per-tenant
predicates are therefore available to every Cedar surface, and the RFC 8693 broker cutover
(AD-127) stays deferred rather than being promoted as a prerequisite.

Validated on dev 2026-08-19 in `LOG_ONLY` against `dev-receiving-gateway-rodcyvdw7h`, using the
`po_export` M2M client. Three policies, one token, each confirmed `ACTIVE` before its request:

| Probe | `when` clause | Decision |
| --- | --- | --- |
| A | `principal.hasTag("tenantId")` | **Allow** |
| B | `hasTag && getTag("tenantId") == <bound tenant>` | **Allow** |
| C | `hasTag && getTag("tenantId") == <wrong tenant>` | **Deny** |

C is the control: it discriminates a real tag lookup from a policy that was never evaluated.
The live policy was reverted to `infra/policies/po_receiving.cedar` immediately afterwards.

Two constraints came out of the same exercise and bind all future Cedar work here:

- **`getTag` requires a `hasTag` guard.** Cedar cannot prove tag-access safety across the
  Gateway's generated schema (`AgentCore::OAuthUser`, `AgentCore::IamEntity`,
  `AgentCore::UnauthenticatedUser`), so a bare `getTag` is a hard validation error.
  `validationMode = IGNORE_ALL_FINDINGS` (AD-52) suppresses Cedar *Analysis findings* only — it
  does **not** suppress parse/validation errors.
- **A failed policy update was silent.** `create_policy`/`update_policy` return before validation
  completes; an invalid policy settles into `UPDATE_FAILED` while the engine keeps serving the
  *previous* policy. The provisioner reported success and Terraform saw a clean apply, so a
  no-op deploy was indistinguishable from a real one. `scripts/manage_policy_engine.py` now
  polls the policy to `ACTIVE` and exits non-zero with the `statusReasons` attached.

This second point is a correction to AD-52's provisioner contract, not merely a script bug: any
gate that reads decision telemetry after a policy change is meaningless unless the change is
confirmed to have landed. The first probe run in this exercise produced exactly that false
reading — an `Allow` from the *old* scope-based policy.

## Alternatives Considered

- **Infer the answer from the token.** Rejected: the token demonstrably carries a top-level
  `tenantId` claim, but claim-to-tag projection is AgentCore's behaviour, not Cognito's — the
  token proves nothing about what Cedar sees.
- **Promote the RFC 8693 broker cutover (AD-127) first.** Rejected: that is the remedy for the
  *negative* outcome. Building it before knowing the answer would have been unnecessary work,
  and the probe cost half a day against the broker's multi-day cutover.
- **Test with `cedarpy` alone.** Rejected: `cedarpy` exercises evaluation logic against a
  hand-written entity, which is precisely the assumption under test. It also cannot see the
  Gateway's generated schema — the same blind spot that sank an earlier per-tool design.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Per-tenant predicates available to every Cedar surface | A `hasTag` guard is now mandatory boilerplate on every tag access |
| Provisioner failures are loud; gates that read telemetry are trustworthy | Policy deploys are slower — each now blocks on a status poll |
| AD-127's broker cutover stays deferred | The dev policy was briefly mutated to probe it (`LOG_ONLY`, reverted) |

The `sub` is still the App Client id, so `principal` remains a machine identity — `tenantId`
adds a *tenant* dimension, not a *user* one. Segregation-of-duties policy (approver ≠ requester)
still needs either the AD-127 delegation chain or a user-token surface such as the Node 6
approval gate.

## Results

- `scripts/manage_policy_engine.py` — `_wait_policy_active` gates every create/update.
- `scripts/tests/test_manage_policy_engine.py` — pins the regression: `UPDATE_FAILED` must exit
  non-zero and surface its reasons.
- `infra/policies/po_receiving.cedar` — unchanged; the probe was run from a scratch copy so the
  file's `filemd5` Terraform trigger never moved.
- **The wait-for-settle rule generalized within a day.** PR #326's dev deploy then failed in
  `terraform_data.ingest_gateway_observability`: `CreateDelivery` for the XRAY destination was
  rejected because the *same apply* had enabled CloudWatch Transaction Search ~68s earlier and
  the account-level switch had not propagated — eventually-consistent, so a `depends_on` edge
  would not have helped. PR #327 added `_wait_trace_destination_active` to
  `scripts/manage_gateway_observability.py`, the same shape as `_wait_policy_active`. Treat this
  as the standing rule for AD-52's SDK provisioners: **an AgentCore/CloudWatch control-plane call
  that returns is not a call that has taken effect — poll for the settled state before depending
  on it.** PR #327 also fixed dev-deploy.yml retrying a *saved* plan, which made attempts 2–5
  fail instantly with "Saved plan is stale" instead of retrying anything.
- Phase 1 of the rollout shipped on this basis: `infra/policies/pr_ingest.cedar` +
  `test_pr_ingest_policy.py` + `terraform_data.ingest_policy_engine`, live in `LOG_ONLY` on the
  ingest Gateway. Its per-tenant clause (`hasTag("tenantId")`) is the first policy predicate this
  ADR's finding made expressible — it fails closed on a token minted without a tenant binding,
  which the scope check alone cannot detect.
- Unblocks per-tenant Cedar predicates on the ingest Gateway and on Gateways fronting the three
  currently ungated MCP runtimes (`dynamodb_master_data`, `step_functions_orchestrator`,
  `tenant_mdm_emulator`), which today are invoked directly via boto3 with no authorizer,
  no interceptor and no Cedar at all.
- Any future per-tool policy must still validate against the Gateway's *generated* schema before
  apply; `cedarpy` tests alone do not cover it (AD-39).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
