# AD-048 — Fail-Fast on System-Config Unavailability

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-48 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-25, AD-45, AD-46, AD-49, AD-63, AD-95

## Context

All dynamic configuration — model IDs, governance thresholds, resilience parameters — lives in `{env}-system-config`. Seven resilience patterns (AD-45) are themselves config-driven, creating a dependency: the agent cannot know its retry budgets, circuit-breaker thresholds, or governance limits without reading the config table. AD-46 establishes an availability-first degradation posture for most dependencies (quality-enhancers fail open), but that principle cannot extend to the config plane: an agent that runs on guessed governance thresholds may silently approve spend above its limit, disable guardrails, or apply wrong risk weights, with no visible signal of the corruption.

## Decision

If `{env}-system-config` is unreachable at agent instantiation after SDK-level retries, the agent fails fast with a clear error. DynamoDB is the single source of configuration truth; no stale cache, no hardcoded fallback, no silent degradation. The operational response is to restore DynamoDB availability; the failed instantiation surfaces through the standard DLQ → `REQUIRES_ATTENTION` path. This is a deliberate inversion of AD-46's availability-first stance, scoped specifically to the config plane.

## Alternatives Considered

- **Last-known-good sidecar cache (the retired AppConfig model).** Rejected: reintroduces the "running on stale config" risk this decision exists to eliminate; AppConfig was retired at platform v1.7.0 precisely for this reason.
- **Conservative hardcoded fallback defaults.** Rejected: "conservative" is not a well-defined property when multiple thresholds interact; any hardcoded set is either too permissive somewhere or too restrictive somewhere else, and there is no signal that the agent is not running on its intended config.
- **Degrade gracefully with a `config_degraded` flag.** Rejected: governance decisions made under unknown config are not recoverable after the fact; the correct outcome is a clean failure, not a completed negotiation with a caveat attached.

## Trade-offs

| Gained | Given up |
| --- | --- |
| An agent never executes on stale or invented governance thresholds — wrong approval limits or disabled guardrails are structurally impossible | Availability during a config-plane outage: the system stops rather than degrades |
| Config-plane failures are loud and surfaced immediately (DLQ → `REQUIRES_ATTENTION`) rather than silently corrupting decision quality | The config plane is now an explicit hard dependency for every agent instantiation |
| Correctness of behaviour is tied to correctness of config — the two cannot diverge silently | A transient DynamoDB blip that outlasts SDK retries will fail negotiations in flight |

This posture is a deliberate contrast to AD-46: for quality-enhancers (memory, Mem0) the system prefers availability; for the config plane, correctness beats availability. The rule has **two bounded, named exceptions**: AD-49 (feature flags fall back to seed-matching safe defaults, the two security-critical flags primary) and AD-95 (standalone A2A agent runtimes fall back for model-tier resolution only on their own init path). Both are scoped narrowly — flags and model-tier respectively — and neither relaxes the fail-fast rule for governance thresholds, temperatures, or model config on the canonical `DynamicAgentFactory` path.

## Results

REQ-R405 and the degradation table (PRD-006 §3.1) mark `DynamoDB system-config` as "fail fast / await recovery." The decision is realized in `DynamicAgentFactory` (AD-25, AD-63) where the fail-fast path surfaces as a construction-time exception. Two carve-outs apply: AD-49 (feature-flag safe defaults — a default is provably safer than a failure for the security-critical flags) and AD-95 (A2A agent model-tier fallback — the standalone A2A runtimes resolve their model at init off the factory path, where a transient config read must not block a cold start). Both are recorded as bounded exceptions in PRD-010 §3.2 / REQ-C003; the canonical factory path remains fail-fast for all value config.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
