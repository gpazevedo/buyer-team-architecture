# AD-160 — Oracle Gets an MCP Transport Only Because We Build the Server; the Flip Is a Separate Decision

**Theme:** Integration (Skills, Plugins & Transports)
**Catalog:** AD-160 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-68, AD-97, AD-155, AD-158, AD-161

## Context

The Oracle integration arrived with a transport branch and no transport. `oracle_pr_poller.py` read an `ORACLE_VIA_MCP` flag and `oracle_rest_mcp.py` existed to service it, but there was no server behind either: unlike SAP — where AWS publishes and operates the "AWS for SAP MCP Server" that AD-155's Gateway fronts — **there is no AWS-provided Oracle MCP server**. Anything the flag routed to would have to be built, containerised, deployed and operated by us. The branch was dormant only because the flag defaulted off and every failure fell back to direct HTTP, which is precisely the condition under which a broken transport is indistinguishable from a working system.

That left a real choice rather than a formality. Deleting `oracle_rest_mcp.py` and the flag was legitimate and much cheaper — it would have removed a transport that had never been exercised end to end against a real server. Keeping it meant accepting the single largest item in the Oracle remediation plan: an image, an AgentCore Runtime, a Gateway, a Cognito client per tenant, a Cedar policy and a tenant binding, all of which SAP gets for free from AWS.

A second question sat underneath: *what surface would our server serve?* The first draft of `oracle_rest_mcp.py` had been copied from `sap_odata_mcp.py` and spoke AWS's generic `find_sap_services`/`odata_read` idiom — a catalog to discover, a service name to pin, a generic read tool taking an entity name. That shape exists because AWS's server must serve arbitrary, unknown OData services. Ours does not.

## Decision

**Keep the MCP transport, and build the server over the spec-generated tool surface (AD-161) rather than a generic read/write idiom.** `buyer-team-oracle`'s `mcp_server/server.py` serves one MCP tool per spec operation — 23 of them — straight out of `generated/oracle_tools.py`. `orchestrator/oracle_rest_mcp.py` therefore calls Oracle operations *by name* (`getPurchaseRequisitions`, `getPurchaseRequisition`, `getSuppliers`, `getPurchaseOrders`) and gets Oracle's own REST envelope back. There is no catalog to discover, no service name to pin, and the spec is the contract on both sides of the wire.

Three structural consequences, each a deliberate divergence from the SAP twins and each recorded in the Terraform that implements it:

1. **No S3 custom catalog.** `sap_mcp.tf` provisions one because AWS's server discovers OData services at runtime and must be told which. Ours has one tool per operation and nothing to discover.
2. **No REQUEST interceptor.** `sap_gateway_interceptor` exists to overwrite `arguments.service_name` so one shared runtime can serve many tenants' SAP systems (AD-155). Our runtime has a single `ORACLE_BASE_URL` and no per-call selector, so **a second Oracle tenant is a second Runtime, not a map entry**. Cedar plus the JWT is therefore the Gateway's entire authorization chain, and `_ORACLE_AUTHZ_DENIAL_MESSAGES`' "No Oracle service bound to tenant" can never fire — kept as forward-compat, not pretended to be live.
3. **Own-account ECR pull.** SAP's policy is a cross-account grant on AWS's registry; ours is the ordinary same-account shape every other runtime uses.

**And the flip is its own decision, taken after a live proof rather than a green deploy.** Provisioning the Gateway and routing the pollers' reads through it were kept as two steps. `var.oracle_via_mcp` stayed `false` through the Terraform that created the Runtime and Gateway, and moved to `true` only once a `tools/list` had come back over the Gateway with the generated tool names, and `oracle_rest_mcp`'s own reads had matched the direct-HTTP path row for row through a per-tenant client. Direct HTTP remains the automatic fallback on any MCP exception either way — which is exactly what would have made a premature flip invisible.

## Alternatives Considered

- **Delete the MCP branch entirely (option B of the plan's Phase 5).** Genuinely cheaper and was recommended in the plan absent a deliberate demonstration goal: drop `oracle_rest_mcp.py`, the `ORACLE_VIA_MCP` branch, `is_oracle_mcp_authz_denial`, and their tests. Rejected because Oracle-over-MCP *is* a demonstration goal here — showing the Gateway/Cedar/Cognito pattern holds for an ERP AWS does not supply a server for is the point, not an accident.
- **Serve AWS's generic `find_sap_services`/`odata_read` surface from our own server.** Rejected: it would reimplement a discovery step to solve a problem we do not have (one known base URL, one spec), and would throw away the generated surface's main property — that the spec is enforced identically on both transports.
- **Keep the flag but leave it permanently off ("code-complete, deliberately unrouted").** Rejected as the worst of both: all the code and none of the proof, which is the state the remediation plan existed to end.
- **Per-tenant routing inside one runtime via an interceptor, mirroring AD-155.** Rejected for now: SAP needs it because AWS's shared server takes a `service_name` argument worth pinning. With a single `ORACLE_BASE_URL` there is nothing to rewrite, and a per-tenant Runtime is both simpler and more strongly isolated. Revisit when a second Oracle tenant is real.

## Trade-offs

| Gained | Given up |
| --- | --- |
| An MCP transport for an ERP AWS provides no server for — the Gateway/Cedar/Cognito pattern demonstrated end to end on a runtime we own | We now operate that runtime: an image (ARM64, `:8000/mcp`), a Runtime, a Gateway, a Cedar policy and a per-tenant client — everything the SAP path gets for free |
| One spec-generated contract across both transports; no catalog, no service-name pinning, no interceptor to keep in sync | Multi-tenancy is per-Runtime, not per-map-entry; a second Oracle tenant costs a deployment, and one branch of the authz-denial classifier is permanently unreachable |
| The flip is gated on a live round trip, so the silent direct-HTTP fallback cannot disguise a dead transport | Two transports remain live and must keep agreeing; `list_open_prs` already sends its `q` filter unquoted where the direct-HTTP poller quotes it — the emulator accepts both, a real pod may not |

## Results

Landed as `buyer-team-oracle#4` (`mcp_server/server.py` over the 23 generated tools, stdio + streamable-HTTP) and impl PR #473 (`oracle_rest_mcp.py` rewritten onto the generated tool names; the copied service-discovery shape, `ORACLE_MCP_SERVICE_NAME`, `find_or_create_po` and `close_requisition` deleted). Deployment followed in `buyer-team-oracle#5` (`Dockerfile.mcp`) and impl PR #474 (`infra/oracle_mcp.tf`, `oracle_mcp_gateway.tf`, `policies/oracle.cedar`, per-tenant Cognito client and DynamoDB tenant binding).

That deploy was `READY` and answered nothing: `container_uri` fell back to impl's `git_sha` and `dev/oracle-mcp` in ECR was empty, because the image push is a manual script nobody had run — every call returned JSON-RPC `-32010 "Runtime health check failed"`. Fixed and generalised in impl PR #475 (`infra/tests/test_sibling_repo_image_pins.py`: any image a `scripts/push_*.sh` builds from a sibling checkout is tagged with *that* repo's SHA, so impl's `git_sha` fallback can never resolve it). Three further defects, each of which would also have deployed clean and never answered, were caught by probing the built container and are now regression tests rather than lore: `Mount("/mcp")` 307-redirects the bare `/mcp` path AgentCore POSTs to (must be `Route`, with a raw-ASGI callable *object*); `--port` defaulted to 8080, AgentCore's *HTTP* contract, where the *MCP* contract is 8000; and `_auth()` took the password from a plaintext env var, i.e. from Terraform state, instead of resolving `ORACLE_BASIC_AUTH_SECRET_ARN`.

Live-verified 2026-09-22 before the flip (impl PRs #476, #477): `tools/list` returns the 23 generated tools; a `tools/call` matches direct HTTP byte for byte; a write round trip proves the container resolved its secret; and `test_oracle_rest_mcp_live.py` drives the production read functions through a real `GatewayClient` on the **per-tenant** client, exercising the DynamoDB tenant binding and Cedar that a gateway-level probe skips. `var.oracle_via_mcp` then flipped to `true` (impl PR #477) and was confirmed live: six consecutive poller invocations, zero errors, zero fallback warnings, the runtime logging `POST /mcp 200 OK`, and all four `dev-buyer-team-oracle-*` alarms — including `oracle-mcp-authz-denied` — at `OK`.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
