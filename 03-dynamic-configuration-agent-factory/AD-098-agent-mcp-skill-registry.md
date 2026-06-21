# ADR-098: Agent/MCP/Skill Registry Config Group

**Status:** Accepted  
**Date:** 2026-06-21  
**PR:** [#41](https://github.com/gpazevedo/buyer-team-impl/pull/41)

## Context

The orchestrator had 6 hardcoded `*_RUNTIME_NAME` module-level constants (e.g.,
`BID_EVALUATION_RUNTIME_NAME = os.getenv(…, "dev_bid_evaluation")`). Each node called
`runtime_arn(RUNTIME_NAME)` directly. There was no single source mapping a logical
agent name to its deployed AgentCore runtime — the mapping was implicit in the
constants and in the seed script's AgentCore Terraform module.

Adding an agent meant touching its node file + the seed script + the Terraform
module, with no runtime-configurable way to remap or discover agents. The
PRD-011 skill runtime also had no discoverability for its MCP servers and skills.

## Decision

Add a **`registry` config group** to the existing `{env}-system-config` DynamoDB
table (5th group, alongside `governance`, `model`, `features`, `external-rates`).
The registry declares:

- **agents** — 7 logical agents, each with `runtime_name`, `protocol` (A2A), `capability`, `model_tier`
- **mcp_servers** — 4 MCP servers, each with `protocol` (MCP), `endpoint`
- **skills** — 3 skills, each with `capabilities` list

A new resolver, `resolve_agent_runtime_name(logical)`, applies 3-tier precedence:

1. **Env override** — `<LOGICAL>_RUNTIME_NAME` environment variable (operational escape hatch)
2. **Registry** — `registry["agents"][logical]["runtime_name"]` from DynamoDB
3. **Legacy fallback** — hardcoded default matching the pre-registry constant (logged at WARNING)

All 6 orchestrator nodes + the accuracy harness call `agent_runtime_arn(logical)`
(`agent_invoke.py`), which composes `resolve_agent_runtime_name` with DNS-resolution
`runtime_arn`.

## Consequences

- **Single source of truth** for agent identity — no runtime names in node files
- **Env override** provides operational flexibility without a config-plane write
- **Legacy fallback** ensures the code builds and runs before the seed populates the registry
- `load_registry_config()` fails fast on DynamoDB unavailability (consistent with AD-48),
  falling through to the legacy default at WARNING level
- Registry is seeded via `scripts/seed_test_tenant.py` alongside the 4 other config groups
- The `registry` item's `config_json` blob is structured, validated at read time
