# AD-045 — Seven Core Resilience Patterns, All Config-Driven

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-45 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-48, AD-13, AD-46, AD-50

## Context

The system must operate correctly under partial failure — supplier MCP unreachable, Bedrock throttled, network partitions — without losing negotiation state, duplicating side effects, or violating governance. Before standardizing, each outbound call site could apply different retry counts, miss a circuit breaker, or omit idempotency protection. Resilience tuning (retry counts, breaker thresholds, timeouts) must be adjustable in production without a redeploy; hardcoded constants require a code change and a deploy to correct a misconfigured production system.

## Decision

Standardize on seven patterns applied at every outbound call site: retry+jitter, circuit breaker, idempotency, bulkhead, timeout, DLQ, and a shared A2A invocation wrapper (PRD-006 §2.1–§2.7). All tunable parameters (`max_attempts`, `fail_max`, timeouts, bulkhead sizes) are sourced from `{env}-system-config` and adjustable without a redeploy.

## Alternatives Considered

- **Hardcoded resilience constants per call site.** Rejected: production tuning requires redeploys; constants drift per call site and do not share a common vocabulary.
- **Library-default retry and timeout settings.** Rejected: defaults are not sized for the platform's latency profile and provide no config-plane observability.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A uniform, named vocabulary applied at every call site | Config-as-data means resilience behaviour depends on the config plane being available and correct |
| Operators retune `max_attempts`, `fail_max`, timeouts, and bulkhead sizes live without shipping code | Every outbound call must route through the shared wrappers — direct SDK use is prohibited |
| Retry exhaustion routes deterministically to DLQ → REQUIRES_ATTENTION | Discipline cost: new call sites must be wired correctly or the pattern is silently absent |

The config dependency is exactly why AD-48's fail-fast rule exists: if system-config itself is unreachable, the system must not run on guessed resilience parameters.

## Results

The shared synchronous `invoke_agent_runtime` wrapper (AD-13, §2.7) composes retry + A2A circuit breaker + timeout for every agent call and additionally emits the per-tenant `agentcore.session_seconds` cost signal, centralizing resilience and cost attribution at one chokepoint. Retry exhaustion routes to DLQ → REQUIRES_ATTENTION. Bulkhead sizes and all other parameters are overridable from `{env}-system-config` at runtime (AD-48 guards that plane). The graceful degradation policy for each dependency type is defined in AD-46; per-operation bulkhead caps are specified in AD-50.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
