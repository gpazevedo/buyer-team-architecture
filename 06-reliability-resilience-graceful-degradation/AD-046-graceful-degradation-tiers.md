# AD-046 — Graceful Degradation Tiers; Memory Failures Never Block

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-46 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-20, AD-48, AD-74, AD-47

## Context

Some dependencies are essential to correctness (DynamoDB system-config, negotiation state) and some are quality-enhancing (AgentCore Memory, Mem0, supplier MCP). Without an explicit tiering policy, each developer decides independently whether a failing dependency should halt a negotiation or degrade gracefully — producing inconsistent behaviour and unobservable quality drops. Treating a quality-enhancer's outage as a hard failure would needlessly halt negotiations; treating an essential dependency as gracefully degradable would allow the system to run on stale or absent configuration.

## Decision

Define a per-dependency degradation tier. For non-essential dependencies, prefer availability over completeness: Memory/Mem0 failures set `memory_degraded=true` and relax quality thresholds but never block a negotiation; supplier MCP circuit-breaks continue with remaining suppliers; Bedrock throttling queues and backs off. Essential dependencies (system-config) are explicitly excluded from this tier and fail fast (AD-48). All degradation events emit `procurement/resilience` metrics with `degradation_type`, `fallback_used`, and `tenant_id` dimensions.

## Alternatives Considered

- **Treat all dependency failures as hard failures.** Rejected: needlessly halts negotiations when only a quality-enhancing dependency is unavailable.
- **Treat all dependency failures as gracefully degradable.** Rejected: allows the system to run on absent or stale configuration, which AD-48 explicitly forbids.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Negotiations keep running through Memory/Mem0 outages and supplier MCP circuit-breaks | Decision quality degrades silently unless the `memory_degraded` signal is surfaced and acted upon |
| Availability is preserved where correctness does not strictly require the dependency | Requires observable degradation signals and a no-auto-award-below-relaxed-thresholds guard to prevent a degraded run from making an irreversible bad call |
| The boundary between essential and non-essential dependencies is explicit and testable | Two classes of dependency must be maintained as the system grows; misclassifying a new dependency has serious consequences |

## Results

The per-dependency degradation tiers table is defined in PRD-006 §3.1. The `memory_degraded` flag (AD-20) propagates through the negotiation state and feeds the no-auto-award guard at Node 5. All degradation events emit `procurement/resilience` metrics. This decision pairs with AD-72's independently-degrading two-tier memory and AD-74's four Mem0 integration points, each of which is explicitly non-blocking. AD-47 applies the same availability-first principle to Kraljic Classifier failures via a rule-based fallback. AD-48 is the named exception: config-plane failures are excluded from this tier and fail fast.

**Timeout-ordering invariant (added 2026-06-20, PR #39).** Graceful degradation of the A2A agent path only fires if the dependency timeout is ordered **strictly below the executor's own budget** — otherwise the executor dies before the `try/except → deterministic fallback` runs. This silently broke on the A2A path: the boto read timeout was 150s while the agent-node Lambda budget is 120s, so a cold/hung AgentCore runtime hard-timed-out the Lambda with no fallback (observed right after a VPC restore). Fixed by wiring the A2A read timeout to governance `a2a_agent_call_seconds` (150→**100**, below 120) with boto retries off and the §2.1 retry `max_attempts` 3→1 (one per-call timeout + the node fallback must fit the budget; retrying a hung agent in-budget is impossible). The invariant — *every external-call timeout must sit below the timeout of whatever executor would otherwise be killed first* — is now a cross-PRD row (PRD-006 §2.5, owner; cited by PRD-007 §3.4 Lambda budgets). Cold starts are paid up front via `orchestrator/warm_runtimes.py` so the rare hang is a genuine failure, not a first-call artifact. The shared invoke wrapper (AD-13) is where the timeout is carried.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
