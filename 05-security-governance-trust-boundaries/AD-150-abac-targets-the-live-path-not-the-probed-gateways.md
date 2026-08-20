# AD-150 — ABAC Targets the Live Path, Not the Probed Gateways; Re-Hosting Stays Deferred

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-150 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-38, AD-39, AD-52, AD-147, AD-148, AD-149

## Context

REQ-S704–S706 (per-request ABAC) is built but inert: `lambdas/gateway_interceptor/handler.py`
already does `AssumeRole` + tenant-tag session-tagging and fails closed on error, gated by an
`ABAC_TOOL_ROLES` map that is empty on every Gateway. The interceptor's own docstring gives the
reason: "the ingest Gateway fronts the skill ingest tool, whose ARNs are not tenant-partitioned."
That framed the natural next step as picking a *different*, already-tenant-partitioned Gateway to
populate the map on first — `tenant_mdm` or `dynamodb_master_data` looked like the candidates,
since their MCP servers key every table by `tenant_id`.

Two checks against the live code overturn both halves of that framing.

**First, the docstring's premise is false.** `skill_runtime/server.py` — the runtime actually
served behind both the ingest and receiving Gateways — writes to `tenant_id`-keyed tables
(`{ENV}-test-tenant-master-purchase-requisitions`, keyed `tenant_id`+`requisition_id`) and to
`tenant_id#`-prefixed composite-key tables (`{ENV}-requisitions` at `pk = "{tenant_id}#{rid}"`,
`{ENV}-negotiations` similarly). `ingest_purchase_requisitions`, `receive_purchase_order`,
`acknowledge_purchase_order`, `reject_purchase_order`, and `get_order_history`/`get_order_detail`
all take `tenant_id` and resolve to these tables. The one real exception is `auto_start_workflow`'s
`states:StartExecution` call, which targets a single shared state machine ARN with no per-tenant
IAM handle — so a role scoped to this tool's DynamoDB legs cannot also scope that call; the tool as
a whole gets partial, not total, IAM isolation. That is a real constraint, but it is not "not
tenant-partitioned."

**Second, the candidates the framing pointed to have no live caller.** AD-148's addendum
(2026-08-19) already established this for a different reason (Cedar per-tool gating), and it holds
identically for ABAC: `skill_runtime/server.py` never calls `step_functions_orchestrator`,
`tenant_mdm_emulator`, or `dynamodb_master_data` — it does its own direct `boto3` reads and
`Step Functions` calls instead. `skills/test_tenant_master/` and `skills/integration/`, the tree
that *would* call them, is imported by nothing live. Wiring `ABAC_TOOL_ROLES` onto any of those
three Gateways would scope credentials for traffic that structurally cannot arrive — a real IAM
change protecting nothing, with the same fail-closed blast radius (AD-149's degrading-caller shape
does not apply here since there is no caller to degrade, but a bad `${aws:PrincipalTag/tenant_id}`
substitution would still be indistinguishable from correct in `LOG_ONLY`-equivalent telemetry,
because no request ever exercises it).

`dynamodb_master_data` is additionally excluded from this mechanism specifically, independent of
the live-caller finding: AD-148 already recorded that its Gateway carries no REQUEST interceptor
(its tools take a generic `table`/`key` shape, not a uniform `tenant_id` argument), and the
interceptor is what calls `_tenant_credentials()` — so there is no code path that could hand it
`tenant_credentials` regardless of what `ABAC_TOOL_ROLES` said.

## Decision

**ABAC's first Gateway is the one carrying live traffic: ingest and/or receiving, fronting
`skill_runtime/server.py` — not `tenant_mdm` or `dynamodb_master_data`.** The tenant-partitioned
tools to enumerate in Phase 1's `ABAC_TOOL_ROLES` map are `ingest_purchase_requisitions`,
`receive_purchase_order`, `acknowledge_purchase_order`, `reject_purchase_order`,
`get_order_history`, `get_order_detail` — scoped to the DynamoDB legs only. `load_datasets` /
`validate_datasets` / `reset` stay out: their S3 leg reads a single shared prefix
(`s3://{ENV}-procurement-data-{account_id}/raw`, not per-tenant), so an ABAC role would only cover
part of what they do, the same partial-coverage shape as `auto_start_workflow`.

`tenant_mdm_emulator` and `step_functions_orchestrator` remain valid ABAC candidates **once**
`skills/test_tenant_master/`/`skills/integration/` is wired live (a separate, larger decision this
ADR does not make) — their tools are genuinely tenant-partitioned and interceptor-fronted, so the
map entries this ADR skips for them today are cheap to add later. `dynamodb_master_data` stays
excluded regardless, for the interceptor-absence reason above.

This also resolves Gap 3's open question — re-host to `mcp.lambda` for per-tool Cedar, or accept
ABAC as the per-tenant control and leave Cedar coarse (AD-148's deferred alternative). **Re-hosting
stays deferred**, now for a sharper reason than "disproportionate for this phase": two of the three
servers AD-148 costed the re-host for have no live caller, and the one live, tenant-partitioned
path (`skill_runtime`) is an AgentCore Runtime for reasons unrelated to Cedar (it is the deployed
Skill entry point), so re-hosting it is a materially bigger change than re-hosting an idle MCP
server. ABAC delivers the per-tenant control the coarse Cedar gate cannot on the path that actually
carries traffic; Cedar stays the deterministic outer gate (scope + `tenantId`-tag presence) on all
five Gateways, exactly as AD-148 describes.

## Alternatives Considered

- **Wire ABAC onto `tenant_mdm` first, as the original framing suggested.** Rejected: protects no
  live traffic today (AD-148's addendum); the engineering and IAM-blast-radius cost buys nothing
  until the dead call tree is wired live.
- **Wire ABAC onto `dynamodb_master_data`.** Rejected outright, not merely deferred: no interceptor
  fronts that Gateway, so `tenant_credentials` can never reach it through this mechanism regardless
  of `ABAC_TOOL_ROLES` content.
- **Treat `ingest_purchase_requisitions` as fully ABAC-eligible.** Rejected: `auto_start_workflow`'s
  `states:StartExecution` has no per-tenant IAM handle on a shared state machine. Scoping the role
  to the DynamoDB legs and accepting the SFN call as outside ABAC's reach is the honest middle
  ground, not a blocker to starting.
- **Re-host `skill_runtime` as a Lambda now, to get per-tool Cedar on the live path.** Rejected for
  this phase: bigger than the MCP-server re-host AD-148 already declined, and ABAC gets most of the
  same per-tenant benefit without a runtime-hosting migration.

## Trade-offs

| Gained | Given up |
| --- | --- |
| ABAC investment lands on the path that actually carries traffic | The originally-suggested "smallest Gateway first" ordering is abandoned — ingest/receiving are not the smallest, they are the live ones |
| `dynamodb_master_data` is permanently ruled out rather than left as an open candidate | One fewer Gateway this mechanism can ever cover without a separate interceptor decision |
| `auto_start_workflow` / dataset-loading tools get an honest partial-coverage label instead of a false "not tenant-partitioned" one | Those tools still carry a real, uncloseable IAM gap (shared state machine, shared S3 prefix) that ABAC alone does not fix |
| Re-host stays deferred with a sharper, evidence-based reason | Per-tool Cedar on the live path remains unavailable until re-hosting is separately decided |

## Results

Supersedes the "ingest is not tenant-partitioned" premise in
`lambdas/gateway_interceptor/handler.py`'s module docstring and `infra/gateway.tf`'s header
comment — both corrected in place.

- `skill_runtime/server.py`, `skills/test_tenant/test_tenant_skill/po_receiving.py`,
  `skills/test_tenant/test_tenant_skill/mcp_clients.py` — `tenant_credentials` threaded from
  each of the six eligible tools down through their DynamoDB calls; `_ddb_resource`/`_ddb_client`
  build a per-call, uncached client from the temporary credentials when present, the shared
  execution-role client otherwise.
- One further scoping correction found while wiring this: `_cancel_domain_requisition`'s
  `requisition_index` GSI query (keyed by `requisition_id`, `ALL` projection) cannot be
  tenant-scoped via `dynamodb:LeadingKeys` — a different partition key — so it stays on the
  execution role, the same treatment as `_start_workflow`. Only `_get_master_pr` /
  `_claim_master_pr` / `_upsert_domain_requisition` (the NEW-status ingest path) get
  `tenant_credentials`.
- `skill_runtime/tests/test_abac_tenant_credentials.py` — proves the credentials actually reach
  boto3 as the AWS identity for every eligible tool, and that the SFN/GSI legs above do not
  receive them.
- `infra/gateway.tf` — `aws_iam_role.ingest_abac` / `aws_iam_role.receiving_abac`, each scoped by
  `dynamodb:LeadingKeys` to exactly the tables its tools touch; the interceptor's own role gains
  `sts:AssumeRole`/`sts:TagSession` scoped to just these two ARNs; `ABAC_TOOL_ROLES` populated on
  the shared interceptor Lambda. `terraform validate` passes; **not applied** — dev apply and the
  Phase 3 burn-in are a separate, deliberate step.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
