# AD-025 — DynamicAgentFactory and Config-as-Data (Fail-Fast, No Fallback)

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-25 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-48, AD-49, AD-63, AD-65, AD-28

## Context

Agent parameters — model ID, temperature, max tokens, evaluation thresholds — must be tunable per tenant and per environment without redeploying code. The earlier design used AWS AppConfig with sidecar caching and gradual rollout; this was retired at platform v1.7.0 in favor of a single DynamoDB `{env}-system-config` table. A sharper question then surfaced: if config is unreachable at construction time, should the agent fall back to safe hardcoded defaults, or refuse to start? The previous answer was a `_COLD_START_DEFAULTS` fallback (REQ-A006c), which directly contradicted the single-source-of-truth contract (PRD-010 REQ-C003) and was removed in v1.0.24.

## Decision

Every agent is instantiated via `DynamicAgentFactory`, which sources all parameters from `{env}-system-config` at construction time — nothing is hardcoded. Thresholds resolve in two stages: per-tenant DynamoDB overrides, then the system-config profile selected by SK (`default` / `profile#conservative` / `profile#aggressive`). Config is read once at construction (no polling, no mid-session change); changes take effect on the next instantiation (REQ-A006/A705). If a required config item is unreachable after SDK-level retries, the factory fails fast with a clear error — it does not substitute hardcoded config, stale cache, or conservative defaults, and it does not construct a degraded agent.

## Alternatives Considered

- **`_COLD_START_DEFAULTS` fallback.** Rejected: allows an agent to run on guessed governance values, silently corrupting decision quality — and was a direct contradiction of PRD-010 REQ-C003; removed in v1.0.24.
- **AppConfig with sidecar caching and gradual rollout.** Rejected: added a sidecar, polling, and operational surface without solving the "running on stale config" risk; retired at platform v1.7.0 (replaced by AD-63).
- **Last-known-good in-process cache.** Rejected: reintroduces the stale-config risk with no visible signal that the values are not current.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Per-tenant tuning and config changes with no redeploy | Read-once semantics mean no mid-session config change; tuning changes wait for the next instantiation |
| A single, unambiguous source of configuration truth — an agent never runs on guessed values | The config plane becomes an agent availability dependency: an unreachable required item is a hard error → DLQ → `REQUIRES_ATTENTION` |
| `DynamicAgentFactory` is the single request-assembly point, enabling it to own the cache-prefix invariant (AD-65) | The fail-fast contract required removing a long-standing fallback; a contradictory `_COLD_START_DEFAULTS` survived from the AppConfig era until v1.0.24 |

## Results

`DynamicAgentFactory` is implemented in PRD-003 §3.1 and PRD-010 §3; the fail-fast recovery contract is REQ-C003. A failed construction surfaces through the standard agent invocation-failure path (DLQ → `REQUIRES_ATTENTION`) rather than silently degrading decision quality. The decision enables AD-65 (factory as single cache-checkpoint owner) and is guarded by AD-48 (fail-fast rule for the config plane) with one deliberate exception: all `features`-group flags fall back to seed-matching safe defaults when config is unreachable (AD-49), with `galileo_protect_enabled` and `kraljic_classification_override_enabled` → `false` as the primary load-bearing case — a disabled flag means more enforcement, not less.

### Results — as built (2026-06-21)

The factory shipped **scoped to model-ladder + cache-prefix**, not "all parameters". This is a deliberate narrowing of the original decision, settled once governance resolution had already moved into the orchestrator:

- **`DynamicAgentFactory` (in `impl/packages/buyer_agent_core/`, delivered via the `agent-base` image) owns exactly two things:** model-id resolution from the tier ladder (AD-95) and the cache-prefix invariant (AD-65/AD-28). It does **not** read thresholds or feature flags — agents receive governance as a per-request A2A payload and read **no** system-config beyond the model ladder. The earlier spec implying the factory owns thresholds/flags is superseded.
- **The agent boot path is therefore never fail-fast.** The only config read at construction is the model ladder, which is *never-raise* by contract (live `tiers` → `BEDROCK_MODEL_ID` → in-image seed mirror; AD-95). A placement guard test (`test_factory_cache.py::test_boot_path_never_fails_on_unreachable_config`) keeps it that way. The AD-48 fail-fast contract and the two-stage resolution (AD-64) now apply to the **orchestrator's** governance reads (`graph_common.load_governance_config`, `Unreachable` vs `Missing` split), not to agent construction — which is where the recurring "governance blocks dropped from `default`" failure actually needs to fail fast → DLQ → `REQUIRES_ATTENTION`.
- The 7 per-agent `config.py` + `observability.py` duplicates are removed; each agent is now a thin `agent_spec.py` (declares tier/prompt/tools/plugins/schema/card) + `agent.py` (≈30 lines) over the shared package, byte-identical model resolution and cache prefix across all of them.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
