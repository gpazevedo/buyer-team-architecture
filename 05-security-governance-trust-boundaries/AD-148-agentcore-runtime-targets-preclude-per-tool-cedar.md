# AD-148 — AgentCore Runtime Targets Preclude Per-Tool Cedar; Gate Coarsely Now, Re-Host to Go Finer

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-148 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-38, AD-39, AD-52, AD-146, AD-147

## Context

The Cedar rollout's phase 2 was to put Gateways in front of the three MCP servers that had none
(`step_functions_orchestrator`, `tenant_mdm_emulator`, `dynamodb_master_data`), and phase 3 was to
use those Gateways for per-tool policy — `forbid` on `DeleteItem`/`BatchWriteItem`, permits scoped
per tool. The plan asserted that because these servers already run `server_protocol = "MCP"`,
fronting them with **MCP targets** rather than HTTP-runtime targets was both natural and "the only
route to phase 3."

Two premises behind that turned out to be false, and one adjacent assumption was load-bearing.

First, nothing actually invoked these runtimes. `orchestrator/graph_common.py`'s cited lines are a
logical-name → runtime-name lookup table, not a dispatch; `mcp_runtime_arn()` and
`resolve_mcp_runtime_name()` had zero production callers. The only code that called an MCP server
was `_invoke_mcp` in `skills/test_tenant_master/helpers.py`, which built
`boto3.client("agentcore-gateway")` — not an AWS service — and so raised `UnknownServiceError` at
all ~27 of its call sites. It also unwrapped `result["data"]`, a key no MCP tool produces.

Second, `server_protocol = "MCP"` on a *runtime* describes how that runtime speaks. It does not
make the runtime eligible as a Gateway **MCP target**. Those are unrelated classifications.

## Decision

**An AgentCore Runtime can only ever be an HTTP target, so per-tool Cedar actions are unreachable
for a server hosted that way.** Phase 2 therefore delivers the same coarse outer gate the ingest
and receiving Gateways have — deny-by-default, `orchestrate` scope, plus AD-147's `tenantId`
presence check — and phase 3's per-tool policy requires **re-hosting the server as a Lambda**
(`mcp.lambda` target), not a change to any policy file.

Per the AgentCore Gateway documentation (verified 2026-08-19): HTTP target types "include Amazon
Bedrock AgentCore Runtime agents", while MCP target types are Lambda functions, API Gateway REST
APIs, OpenAPI specifications, Smithy models, and MCP servers — where an MCP-server target requires
an **HTTPS endpoint URL**. An AgentCore Runtime has no such URL; it is reached through
`InvokeAgentRuntime` over SigV4. Only MCP targets generate the per-tool `<target>___<toolName>`
actions Cedar would need. The repo had already recorded half of this constraint from the other
direction — "AgentCore Runtime targets cannot be added to MCP protocol type gateways" — without
drawing the per-tool consequence.

Two further decisions follow from building it:

- **Order by what gating actually buys, not by blast radius.** `step_functions_orchestrator` goes
  first: smallest surface (2 tools), one destructive (`cancel_negotiation`), and its tools take
  `tenant_id` under `require_tenant()` — so the Gateway upgrades it from a trusted request body to
  a JWT-verified claim, exactly the migration `mcp_servers/shared/tenant.py` anticipates.
- **`dynamodb_master_data` must NOT reuse the shared REQUEST interceptor.** That interceptor sets
  `arguments.tenant_id` unconditionally on every `tools/call`, and master-data's tools
  (`Query`/`PutItem`/`GetItem`/…) accept no such parameter — injecting it would break every call.
  Its scoping boundary is `ALLOWED_TABLES`, not tenancy, so it gains only the coarse gate.

## Alternatives Considered

- **Front the runtimes with MCP targets as planned.** Rejected: not expressible. An MCP-server
  target needs an HTTPS endpoint the runtime does not have.
- **Re-host all three as Lambdas now, to reach phase 3 immediately.** Rejected for this phase: it
  is a migration of three working servers off AgentCore Runtime — new packaging, new IAM, new cold
  start behaviour, and a rewrite of AD-146's `build_mcp_app` entry point — disproportionate to
  doing it before the coarse gate exists. Recorded as the actual prerequisite for phase 3.
- **Skip phase 2 since nothing calls these runtimes.** Rejected: they are deployed and `READY`,
  reachable by any principal holding IAM `InvokeAgentRuntime`, and `_invoke_mcp` is now fixed, so
  callers are about to exist. A front door built before the traffic arrives is cheaper than one
  retrofitted after.
- **Leave `_invoke_mcp` dead and delete its call sites.** Rejected: the three servers implement
  real behaviour the Skill runtime needs (`start_negotiation_workflow` is how ingest triggers
  PRD-002's workflow), so the callers are wanted; the client was simply never made to work.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A real authorization boundary in front of runtimes that had none | Only a coarse one — the body stays opaque, so `cancel_negotiation` and `start_negotiation_workflow` are indistinguishable to Cedar |
| `step_functions_orchestrator` moves from trusted-body to verified-claim tenancy | A per-tenant M2M client per tenant, mirroring `PO_EXPORT_CLIENT_MAP`'s multiplication |
| ~19 of 27 `_invoke_mcp` call sites become functional | 8 remain blocked on two servers that were never built |
| Phase 3's true prerequisite is now known and costed | Phase 3 is further away than the plan claimed |

The coarse gate is worth having on its own terms — it is the same control the receiving Gateway
has carried since AD-39 — but it should not be described as per-tool authorization. The
interceptor and the servers' own `ALLOWED_TABLES` / `require_tenant()` checks remain the
fine-grained layer, exactly as AD-38 intends.

## Results

- `skills/test_tenant_master/helpers.py` — `_invoke_mcp` rewritten onto real
  `bedrock-agentcore:InvokeAgentRuntime` MCP calls with paginated ARN resolution; returns the
  tool's own value; `McpServerNotDeployed` names the two logical servers (`s3-reader`,
  `dynamodb-domain`) that callers reference but `mcp_servers/` never implemented.
- `skill_runtime/tests/test_mcp_invoke.py` — 8 tests pinning the transport, both response
  encodings, the pagination gotcha, and the named errors.
- `infra/agent_runtimes.tf` — the Skill runtime role gains `InvokeAgentRuntime` scoped to the
  three MCP runtimes, plus `ListAgentRuntimes` for name→ARN resolution.
- `infra/mcp_gateways.tf` + `infra/policies/sfn_orchestrator.cedar` (+ its `cedarpy` tests) — the
  first of the three Gateways, `LOG_ONLY`, with the AD-147 per-tenant clause.
- `infra/modules/security/main.tf` — an `orchestrate` scope and two per-tenant M2M clients, with
  the `by-app-client` binding rows REQ-S708's normaliser fails the mint without.
- Open, and deliberately not closed here: `s3-reader` and `dynamodb-domain` have no
  implementation, blocking 8 call sites (the master-data loaders' S3 reads and ingest's domain
  writes); the remaining two Gateways; and the Lambda re-hosting that phase 3 now depends on.

### Addendum (2026-08-19) — the calling tree is dead, not merely incomplete

Completing phase 2 established that the two missing MCP servers were a symptom of something
larger: **the entire tree that calls them is unreachable.** `skill_runtime/server.py` — the
runtime actually served — imports exclusively from `skills/test_tenant/test_tenant_skill/` and
contains **zero** references to any of the three MCP runtimes. It reads master data with direct
boto3 (`_get_master_pr`) and starts workflows with direct Step Functions (`_start_workflow`).

`skills/test_tenant_master/` and `skills/integration/` are imported by nothing live — only by
each other. `ingest_purchase_requisitions` is defined **twice**, and the live definition is the
one in `server.py`, not the `_invoke_mcp`-based one in `skills/integration/ingest_pr.py`. A
parallel, working implementation of what `s3-reader` / `dynamodb-domain` were meant to provide
already exists in `skills/test_tenant/test_tenant_skill/mcp_clients.py`, backed by boto3.

Two consequences. First, the `_invoke_mcp` repair recorded above is real but currently inert:
it fixed a client in a tree nothing imports, so "19 of 27 call sites now work" means they would
work if something called them. Second, and more importantly for this ADR's decision: all three
Gateways gate runtimes with **no live caller**, so their gates can only be tripped by a
deliberate probe. That does not invalidate them — the runtimes remain reachable by any principal
holding IAM `InvokeAgentRuntime`, which is exactly the exposure a front door closes — but it
does mean **no `ENFORCE` flip should happen until a real caller routes through them**, because
`LOG_ONLY` telemetry from a probe cannot tell you what production traffic would be denied.

The open decision this creates, deliberately not taken here: either wire the live path through
the Gateways (`server.py`'s direct boto3 calls become Gateway calls) and delete the duplicate
tree, or delete the duplicate tree and accept that these Gateways stay probe-only. Building
`s3-reader` and `dynamodb-domain` is not on either path — it would be new infrastructure for
dead code.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
