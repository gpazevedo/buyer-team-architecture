# AD-155 — sapmcp Gateway Tenancy: Pin `service_name`, Not `tenant_id`

**Theme:** Integration (Skills, Plugins & Transports)  **Catalog:** AD-155 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-70, AD-147, AD-148, AD-149, AD-52

## Context

`sap_mcp_gateway.tf` fronts AWS's own AWS for SAP MCP Server — a GA server this repo does not
control the code of — with an AgentCore Gateway. When that Gateway went from a one-off
shape-validation target to a runtime dependency of `sap_pr_poller`/`sap_pr_ingest_handler`/
`po_export_drain` (gap-closure plan Phase 3), its file header reasoned through three ways a
multi-tenant SAP integration could route a call to the right tenant's data, entirely on paper,
and picked **one AgentCore Runtime + Gateway per tenant**. It rejected a shared runtime with one
custom-catalog entry per tenant because "`find_sap_services` has no tenant-scoping argument
today — a caller could pick the wrong tenant's service by mistake with nothing to stop it," and
rejected an interceptor because "AWS's tools would still need a `tenant_id` argument to rewrite,
which none of them have." Until now the Gateway shipped with neither a Cedar policy engine nor a
REQUEST interceptor — the only one of six Gateways in this repo with neither (AD-148's coarse
gate had not yet been extended here) — so tenancy rested entirely on a caller-side assertion,
`_assert_sap_mcp_tenant`, duplicated in three orchestrator modules, that was a no-op whenever its
env var was unset and was swallowed by each caller's own `except Exception` into a direct-HTTP
fallback whenever it did fire. It never actually routed anything; a second tenant reusing this
Gateway had nothing stopping it from reading or writing the first tenant's SAP data.

Both rejections in the original decision answered a related but different question than the one
that mattered. They are true of `tenant_id` — no tool AWS's server exposes
(`find_sap_services`/`get_metadata`/`odata_read`/`odata_create`/`odata_update`/`odata_count`)
takes one — but every data tool takes `service_name`, naming which custom-catalog entry
(`sap_mcp.tf`'s `aws_s3_object.sap_mcp_catalog`) to read or write. That argument is the
tenant-routing key the original analysis was looking for and did not find, because it was
looking for `tenant_id` specifically.

## Decision

**Route by rewriting `service_name`, not by provisioning a Runtime and Gateway per tenant.**
Three pieces, landed together:

1. A Cedar policy engine in `LOG_ONLY` mode (gap-closure plan A.1, already shipped) gates *who*
   reaches the Gateway. An AgentCore Runtime is an HTTP target, so Cedar sees one coarse action
   (`sapmcp___POST:/invocations`) over an opaque body — it cannot see the tool name or
   `service_name`, so it is not where tenant routing can live (the same ceiling AD-148 names for
   every AgentCore-Runtime-backed Gateway).
2. A new REQUEST interceptor, `sap_gateway_interceptor` — a sibling to
   `lambdas/gateway_interceptor/handler.py`, not a reuse of it, because that Lambda
   unconditionally sets `arguments.tenant_id`, which would break every one of these tools —
   decodes the JWT's `tenantId` claim (already validated by the Gateway's CUSTOM_JWT authorizer)
   and overwrites `arguments.service_name` with the catalog entry bound to that tenant, the same
   unconditional overwrite-or-inject semantics AD-70/AD-41 already establish for `tenant_id`
   elsewhere. An unbound tenant is rejected with a 403 before the target ever runs.
3. One custom-catalog entry and one Cognito M2M client per tenant
   (`var.sap_mcp_tenants`/`module.security.skill_sap_client_ids`), so the interceptor has
   something to pin to and each tenant a credential of its own — the pre-token normaliser's
   by-app-client lookup is a hard-fail 1:1 invariant, so one client cannot resolve to two
   tenants.

`find_sap_services` (catalog enumeration) carries no `service_name` argument and is left
unrewritten in this landing: it can leak another tenant's catalog title/URL to a bound caller,
but cannot cause a mis-read, because every data tool's `service_name` is pinned regardless of
what `find_sap_services` returned. Closing that residual leak is out of scope here (gap-closure
plan A.7) pending a live check of that tool's response shape.

## Alternatives Considered

- **One AgentCore Runtime + Gateway per tenant** (the original decision, superseded here).
  Rejected on reconsideration: it does not depend on AWS's server growing a tenant-scoping
  argument, which is the property the original decision valued, but it means N idling runtimes
  and N Gateways to keep in sync with every future change to this file, and it left the actual
  gap — no enforcement mechanism at all — open for as long as this integration had only one
  tenant to hide it.
- **An interceptor rewriting `tenant_id`.** Rejected in the original decision because no SAP
  tool accepts that argument. Still true; the fix was to rewrite a different argument
  (`service_name`), not to abandon the interceptor pattern.
- **Leave the assertion in place and document the gap.** Rejected: `_assert_sap_mcp_tenant` was
  a no-op by default and silently bypassed on any exception, so it protected nothing a
  determined or buggy caller couldn't route around — the same fail-open-on-fallback trap AD-149
  names generally.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A real, fail-closed tenancy boundary on the only Gateway of six that had none | A second Lambda (`sap_gateway_interceptor`) and its own IAM role/policy to operate, distinct from the shared interceptor |
| One shared AgentCore Runtime regardless of tenant count — no N-runtimes scaling cost | `find_sap_services` still leaks other tenants' catalog metadata until A.7 |
| Catalog, Cognito clients, and the interceptor's binding all derive from one map (`local.sap_mcp_tenants`), so they cannot drift from each other | A second tenant still needs a coordinated three-part change (catalog entry, Cognito client, DynamoDB binding row) — cheaper than a new Runtime+Gateway, not free |
| Matches the tenant-predicate-rewriting pattern (AD-70) already established elsewhere in the stack | Cedar's ceiling on an AgentCore-Runtime target (AD-148) means the interceptor, not Cedar, carries the actual routing risk — a bug here is a cross-tenant read/write, not merely a bypassed log |

The Cedar/interceptor split mirrors AD-148's finding for every other AgentCore-Runtime-fronted
Gateway in this repo: Cedar's job stops at the door, and anything finer has to happen after it.
That makes the interceptor the single most safety-critical piece of this decision, which is why
`sap_mcp_policy_mode` staying `LOG_ONLY` (AD-149's precondition, gap-closure plan A.5) applies
here unchanged — an interceptor bug under `ENFORCE` would look identical to a healthy system
until a live cross-tenant read was caught by hand.

## Results

- `lambdas/sap_gateway_interceptor/handler.py` + its test suite — the interceptor itself.
- `infra/sap_mcp_gateway.tf` — `interceptor_configuration` wired onto the Gateway,
  `allowed_clients` and the `skill_sap_tenant_binding` DynamoDB item converted to per-tenant
  `for_each`, and the file's header rewritten to record this decision in place of the one it
  overturns.
- `infra/sap_mcp.tf` (`local.sap_mcp_tenants`) and `infra/modules/security/main.tf`
  (`aws_cognito_user_pool_client.skill_sap` `for_each`) — the catalog and Cognito-client halves
  of point 3 above, both keyed by the same tenant map so they cannot disagree with each other or
  with the interceptor's own binding.
- `orchestrator/sap_pr_poller.py`/`sap_pr_ingest_handler.py`/`po_export_drain.py` —
  `_assert_sap_mcp_tenant` replaced by `_sap_mcp_client_id(tenant_id)`, which resolves and fails
  loud on an unmapped tenant instead of asserting a single configured one.
- Still single-tenant in practice: only the dev tenant is bound today. A second tenant is a
  `var.sap_mcp_tenants` map entry plus a second Cognito client, not a new Runtime or Gateway —
  the scaling property the original per-tenant-Gateway decision was reaching for, now achieved
  without it.
- Open, deliberately deferred: the `find_sap_services` catalog-enumeration leak (A.7), and
  extending this same routing to `skill_runtime/server.py`'s own (still single-client,
  gap-closure plan B.4-scoped) SAP MCP calls.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
