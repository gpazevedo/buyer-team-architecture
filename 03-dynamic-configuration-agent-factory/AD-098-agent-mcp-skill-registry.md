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

- **agents** — 7 logical agents, each with `runtime_name`, `protocol` (A2A), `capability`, `model_tier` *(6 since the `bid_evaluation` agent's retirement — see the 2026-08-03 update)*
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

## Update 2026-08-03 (impl PR #255) — tier 3 is now deliberately incomplete

**Agent count is 6, not 7.** The `bid_evaluation` *agent* was retired 2026-07-06 (PR #151, AD-117); Node 5 scores inline. The registry's `agents` map and `scripts/seed_test_tenant.py` both carry 6 entries.

**The legacy-fallback tier no longer covers every agent, and that is the point.** `_LEGACY_AGENT_RUNTIME_DEFAULTS` in `graph_common.py` still mapped `bid_evaluation` → `dev_bid_evaluation` a month after the runtime was destroyed. Because tier 3 is consulted whenever the registry lookup misses, that one stale entry silently resurrected a dead runtime: a caller got an ARN lookup against something that does not exist instead of a clear resolution error, and the WARNING it logged (`fell through to legacy hardcoded default`) read like ordinary pre-seed behaviour. PR #255 removed the entry, so `resolve_agent_runtime_name("bid_evaluation")` now raises `RuntimeError: Cannot resolve runtime name ... not in registry and no legacy default known`.

This sharpens the consequence recorded above. "Legacy fallback ensures the code builds and runs before the seed populates the registry" holds only while an entry names a runtime that actually exists; once it outlives its runtime, the fallback is strictly worse than no entry at all, because it converts an honest resolution failure into a misleading one and masks exactly the registry-config drift it sits above. Removing a retired agent from this map is therefore part of retiring the agent, not cleanup that can follow later.

**The 3-tier precedence is superseded.** AD-131 inserted tenant variant pins and deterministic `ab_split` between the env override and the registry's base `runtime_name`; the live order in `_resolve` is env override → tenant variant pin → `ab_split` → registry `runtime_name` → legacy default. See AD-131 for the variant tiers, and AD-13's 2026-08-03 update for the unpaginated `list_agent_runtimes` bug that broke the ARN lookup this resolver feeds.
