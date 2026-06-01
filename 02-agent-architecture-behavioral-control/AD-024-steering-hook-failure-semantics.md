# AD-024 — Steering Hook Failure Semantics (8 Hooks, No Retry-Wrap, Fail-Closed)

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-24 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-22, AD-23, AD-16

## Context

Given that guardrails are hooks (AD-23), two failure questions must be answered. First, when a hook rejects a call, should the framework automatically retry it? Second, when a hook itself crashes, should the underlying tool still run? These are different failures — a rejection is policy working correctly, a crash is the guard breaking — and the second question is safety-critical: a `WinnerDisclosureGuard` crash that allows the tool to auto-send would leak winner identity and price to rejected suppliers (REQ-A600).

## Decision

Eight hooks total — seven PRE-CALL and one POST-CALL (`BudgetCeilingGuard` on `check_bid_responses`). Hooks are not wrapped in retry decorators (REQ-A009): a policy rejection propagates immediately so the agent can retry with corrected parameters. A hook crash suppresses the target tool — the tool does not execute, and the exception propagates as a tool error into the standard agent-level retry flow (REQ-A009b). This guarantees fail-closed behavior: a security hook crash never results in a tool running with unredacted or non-compliant inputs. `WinnerDisclosureGuard` was moved POST-CALL → PRE-CALL in v1.0.5 to enforce this guarantee at the point of redaction.

## Alternatives Considered

- **Retry-wrap rejections at the hook level.** Rejected: retrying a policy rejection hides the violation signal from the agent; the agent must see the rejection to attempt a corrected call.
- **Fail-open on hook crash (let the tool run).** Rejected: a crashing security hook (e.g., `WinnerDisclosureGuard`) would allow a tool to execute with unredacted inputs, directly leaking sensitive data — the opposite of the intended guarantee.
- **Post-call placement for WinnerDisclosureGuard.** Rejected: a POST-CALL hook runs after tool execution; if the tool auto-sends before the hook fires, the redaction is too late (corrected v1.0.5).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Fail-closed security — a broken guard blocks the action rather than leaking | A buggy or crashing hook becomes a hard availability stop for its tool |
| Clean separation: "policy says no" (cheap agent retry with fixed params) from "guard broke" (agent-level retry → DLQ) | Persistent prerequisite failure drives the negotiation into the rejection-loop → DLQ → `REQUIRES_ATTENTION` path |
| Early warning before hard failure — `rejection_count > 3 in 30s` alarms before the 5-rejection DLQ threshold (REQ-A951) | Operators must watch and act on that alarm; the alarm is the only signal before escalation |

## Results

The fail-closed guarantee is the headline result and the reason `WinnerDisclosureGuard` was moved POST-CALL → PRE-CALL during the failure audit (v1.0.5) — redaction now provably happens before any potential auto-send. The accepted cost is that availability yields to safety: when TCO or risk assessment is persistently unavailable, the `TCOEnforcementGuard`/`RiskAssessmentEnforcement` rejection loop is the expected escalation path, not a bug, and it parks the negotiation in REQUIRES_ATTENTION (AD-16) for an operator rather than guessing. The `steering.hook.rejection_count` metric and the >3-in-30s alarm (REQ-A951) are direct outputs of this decision.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
