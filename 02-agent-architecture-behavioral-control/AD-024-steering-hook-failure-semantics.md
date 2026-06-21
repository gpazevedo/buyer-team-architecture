# AD-024 — Steering Hook Failure Semantics (7 PRE-CALL GUIDE Hooks + 1 Declarative, No Retry-Wrap, Fail-Closed)

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-24 · **Source PRD:** PRD-003 · **Status:** Accepted (mechanism revised at PRD-003 v1.1.3 — Strands 1.43 reconciliation) · **Related:** AD-22, AD-23, AD-16

## Context

Given that guardrails are hooks (AD-23), two failure questions must be answered. First, when a hook rejects a call, should the framework automatically retry it? Second, when a hook itself crashes, should the underlying tool still run? These are different failures — a rejection is policy working correctly, a crash is the guard breaking — and the second question is safety-critical: a `WinnerDisclosureGuard` crash that allows the tool to auto-send would leak winner identity and price to rejected suppliers (REQ-A600).

## Decision

Eight guards total — seven PRE-CALL steering hooks plus `BudgetCeilingGuard`. As reconciled to the shipped Strands API (`strands-agents>=1.43`, PRD-003 §1.3 v1.1.3), the `strands.vended_plugins.steering` interface exposes only a PRE-CALL `steer_before_tool` handler returning `Proceed` / `Guide` / `Interrupt`; it cannot mutate tool input or results and has no after-tool hook. Consequently the seven steering hooks all operate in **GUIDE** mode (cancel the call + feed corrective guidance so the model recomposes) — the originally-specified PRE-CALL MODIFY and REJECT modes both collapse to GUIDE — and `BudgetCeilingGuard` (the former lone POST-CALL MODIFY on `check_bid_responses`) is realized **declaratively** via response fields (`SpotBidResponse.BidResult.budget_flag` / `budget_excess`) + prompt, not as a hook, because the shipped API cannot mutate a tool result.

Hooks are not wrapped in retry decorators (REQ-A009): a GUIDE cancellation propagates immediately so the agent can retry with corrected parameters. A hook crash suppresses the target tool — the tool does not execute, and the exception propagates as a tool error into the standard agent-level retry flow (REQ-A009b). This preserves fail-closed behavior: a security hook crash never results in a tool running with non-compliant inputs. `steering.action` is emitted as `PROCEED` / `GUIDE` / `INTERRUPT`. The PRD-001 §4.3 Layer-5 guard count is unchanged at 8 (7 steering hooks + 1 declarative).

**Superseded mechanism (v1.0.5 → v1.1.3).** The original design — eight *hooks* with PRE-CALL MODIFY/REJECT modes and a POST-CALL MODIFY `BudgetCeilingGuard`, plus a `WinnerDisclosureGuard` that redacted free text and was moved POST-CALL → PRE-CALL in v1.0.5 to redact before any auto-send — was superseded at PRD-003 v1.1.3 when the model was reconciled to the shipped Strands 1.43 API. `WinnerDisclosureGuard` is now a PRE-CALL GUIDE that asserts a bounded template-render map (rejection notifications are deterministic template renders per AD-93 / PRD-003 §2.7; winner fields are not template parameters), so disclosure is impossible by construction and the redaction-timing concern that drove the POST→PRE move is moot.

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

The fail-closed guarantee is the headline result. It originally rested on moving `WinnerDisclosureGuard` POST-CALL → PRE-CALL (v1.0.5); after the v1.1.3 Strands-1.43 reconciliation it rests instead on disclosure being impossible by construction (deterministic template render, winner fields not parameters) plus GUIDE/INTERRUPT cancellation of side-effecting calls before they fire. The accepted cost is that availability yields to safety: when TCO or risk assessment is persistently unavailable, the `TCOEnforcementGuard`/`RiskAssessmentEnforcement` GUIDE loop is the expected escalation path, not a bug, and it parks the negotiation in REQUIRES_ATTENTION (AD-16) for an operator rather than guessing. The `steering.hook.rejection_count` metric and the >3-in-30s alarm (REQ-A951) are direct outputs of this decision. Realization was verified live (PRD-003 v1.1.3) on `BidConfidentialityGuard` / `EvaluationCompletenessGuard` / `WinnerDisclosureGuard` / `AuctionIntegrityGuard`.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
