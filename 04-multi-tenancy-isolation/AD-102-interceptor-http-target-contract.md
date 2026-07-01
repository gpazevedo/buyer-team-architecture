# AD-102 — Gateway Interceptor Built to the HTTP-Target Contract

**Theme:** Multi-Tenancy & Isolation
**Catalog:** AD-102 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-41, AD-42

## Context

AD-41 requires the Gateway interceptor to read the request body and overwrite `tenant_id` before the target runs. But an AgentCore Gateway's interceptor event shape depends on the **target type** it fronts, and the two shapes are not interchangeable:

- A **native MCP target** delivers `event["mcp"].gatewayRequest` with the JSON-RPC body already parsed into an object.
- An **HTTP target** delivers `event["http"]` with the body as a **base64-encoded string**.

Our `ingest` Gateway does not front a native MCP server — it fronts an **AgentCore Runtime** (the skill MCP server exposed as a Runtime, per the live PR→PO ingest path), which the GA interceptor API classifies as an **HTTP target**. The PRD-005-impl reference interceptor was written against the MCP shape (`event["mcp"]`, parsed body); dropped onto our Gateway it never fires — `event["mcp"]` is absent and the body is an opaque base64 string. Without resolving this, AD-41's tenant-rewrite control would silently no-op on the one Gateway that carries live tenant traffic.

## Decision

Build the interceptor to the **GA HTTP-target contract**: read `event["http"]`, base64-decode the body, parse the JSON-RPC envelope, overwrite `params.arguments.tenant_id` with the JWT-validated claim (AD-42), re-encode, and return a `transformedGatewayRequest` — or a fail-closed `transformedGatewayResponse` (403 missing claim / 500 on throw, target never runs). `pass_request_headers = true` is set so the interceptor can read the `Authorization` header (provider default is `false`). The HTTP-target and MCP-target shapes are recognised as **distinct interceptor code paths** selected by target type; this Gateway uses the HTTP path.

## Alternatives Considered

- **Reuse the PRD-005-impl MCP reference sample as-is.** Rejected: it keys on `event["mcp"]` and a parsed body, so it does not execute on an HTTP/Runtime-backed Gateway — the control would be present in config but inert at runtime.
- **Convert the ingest Gateway to a native MCP target.** Rejected: the ingest path is a deployed AgentCore Runtime fronting the skill MCP server; re-plumbing it to a native MCP target to match the sample re-architects the live ingest path for no isolation benefit.

## Trade-offs

| Gained | Given up |
| --- | --- |
| AD-41's tenant rewrite actually runs (and fails closed) on the Runtime-backed Gateway that carries live ingest traffic | The interceptor body-handling is target-type-specific — a future Gateway fronting a *native MCP* target needs the separate `event["mcp"]` code path |
| Base64 decode → JSON-RPC rewrite → re-encode is cheap and deterministic; no change to the target or its transport | Couples the interceptor to the GA interceptor event schema (`event["http"]` + base64), a platform contract that can evolve and must be re-validated on provider upgrades |

The split is inherent to the AgentCore interceptor API, not incidental: any Gateway's interceptor must match its target type, so an interceptor library serving both kinds branches on target type at the top.

## Results

Realized in `impl/lambdas/gateway_interceptor/handler.py`; attached via Terraform `interceptor_configuration` with `pass_request_headers = true` (provider 6.50 native, no boto3 out-of-band attach). Shipped and live-validated on dev in PR #43 (Increment 2d). It is the concrete realization of AD-41 on the ingest Gateway and pairs with AD-42 (claim normalization) for the `tenantId` it reads. The deferred per-request ABAC of AD-37 would extend this same HTTP-target path when wired.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
