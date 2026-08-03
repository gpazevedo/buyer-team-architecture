# AD-024 — Steering Hook Failure Semantics (6 PRE-CALL GUIDE Guards + 1 Declarative, No Retry-Wrap, Fail-Closed in a Base Class We Own)

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-24 · **Source PRD:** PRD-003 · **Status:** Accepted — the fail-closed guarantee was unrealized until 2026-08-03; see the correction below · **Related:** AD-22, AD-23, AD-16, AD-115, AD-117, AD-133

> **Correction (2026-08-03, impl PR #253) — this ADR's central safety claim was false as written, and the system was fail-*open* on a guard crash from the v1.1.3 Strands-1.43 reconciliation until this date.**
>
> The Decision below asserted that "a hook crash suppresses the target tool — the tool does not execute, and the exception propagates as a tool error into the standard agent-level retry flow." Strands does no such thing. `strands/vended_plugins/steering/core/handler.py` wraps the `steer_before_tool` call in `except Exception`, logs at **DEBUG**, and returns *without* setting `event.cancel_tool` — so a guard that raised let its target tool run unguarded, and did it invisibly at default log levels. The exact scenario this ADR's Context names as the reason the question is safety-critical — a `WinnerDisclosureGuard` crash allowing an auto-send to leak winner identity and price (REQ-A600) — was therefore live for the whole period, not prevented.
>
> The claim was never verified against the shipped framework. It was asserted in the same revision that reconciled the rest of this ADR *to* that framework, which is the specific failure worth remembering: the reconciliation checked the API's shape (what a handler may return) and assumed its failure semantics.
>
> The fix does not restore the claimed mechanism — the framework cannot be patched — but moves the invariant into a layer we own. `buyer_agent_core.steering.GuardHandler` is now the base class for every guard: authors override `check()`, never `steer_before_tool`, and anything escaping `check()` is logged with a stack trace, counted as `governance.hook_error` (dimensioned by guard), and converted into a `Guide` so the call is cancelled instead of allowed. The body below is corrected in place rather than merely annotated — the counts, metric names, and framework behaviour it carried were descriptions of reality, not decisions, and a reader who trusted them would be misled about a safety property.

## Context

Given that guardrails are hooks (AD-23), two failure questions must be answered. First, when a hook rejects a call, should the framework automatically retry it? Second, when a hook itself crashes, should the underlying tool still run? These are different failures — a rejection is policy working correctly, a crash is the guard breaking — and the second question is safety-critical: a `WinnerDisclosureGuard` crash that allows the tool to auto-send would leak winner identity and price to rejected suppliers (REQ-A600).

The second question has a third part that the original decision missed: *who* is responsible for the answer. A framework whose steering handler is fail-open by default does not become fail-closed because an architecture document says it should be.

## Decision

Seven guards total — six PRE-CALL steering-hook classes plus `BudgetCeilingGuard`. (Eight originally: `EvaluationCompletenessGuard` was a seventh hook class, removed along with the `bid_evaluation_llm` runtime at AD-117.) The six classes are registered as eight instances across five agents, because `TCOEnforcementGuard` and `RiskAssessmentEnforcement` are each defined and registered separately on both Bottleneck Negotiation and Strategic Partnership.

As reconciled to the shipped Strands API (`strands-agents>=1.43`, PRD-003 §1.3 v1.1.3), the `strands.vended_plugins.steering` interface exposes only a PRE-CALL `steer_before_tool` handler returning `Proceed` / `Guide` / `Interrupt`; it cannot mutate tool input or results and has no after-tool hook. Consequently the steering hooks all operate in **GUIDE** mode (cancel the call + feed corrective guidance so the model recomposes) — the originally-specified PRE-CALL MODIFY and REJECT modes both collapse to GUIDE — and `BudgetCeilingGuard` (the former lone POST-CALL MODIFY on `check_bid_responses`) is realized **declaratively** via response fields (`SpotBidResponse.BidResult.budget_flag` / `budget_excess`) + prompt, not as a hook, because the shipped API cannot mutate a tool result.

Hooks are not wrapped in retry decorators (REQ-A009): a GUIDE cancellation propagates immediately so the agent can retry with corrected parameters. **A guard crash cancels the target tool** — not because the framework suppresses it, which it does not, but because every guard subclasses `GuardHandler`, whose `steer_before_tool` runs the subclass's `check()` inside a `try` and returns a `Guide` in place of any escaping exception. The guidance deliberately tells the model *not* to retry: a broken guard fails identically on the retry, so the obvious wording would loop to the event-loop cycle cap and burn tokens reaching the same outcome. This is what preserves fail-closed behavior — a security guard's own failure never results in its tool running unchecked. The PRD-001 §4.3 Layer-5 guard count is 7 (6 steering-hook classes + 1 declarative).

**Superseded mechanism (v1.0.5 → v1.1.3).** The original design — eight *hooks* with PRE-CALL MODIFY/REJECT modes and a POST-CALL MODIFY `BudgetCeilingGuard`, plus a `WinnerDisclosureGuard` that redacted free text and was moved POST-CALL → PRE-CALL in v1.0.5 to redact before any auto-send — was superseded at PRD-003 v1.1.3 when the model was reconciled to the shipped Strands 1.43 API. `WinnerDisclosureGuard` is now a PRE-CALL GUIDE that asserts a bounded template-render map (rejection notifications are deterministic template renders per AD-93 / PRD-003 §2.7; winner fields are not template parameters), so disclosure is impossible by construction and the redaction-timing concern that drove the POST→PRE move is moot.

## Alternatives Considered

- **Retry-wrap rejections at the hook level.** Rejected: retrying a policy rejection hides the violation signal from the agent; the agent must see the rejection to attempt a corrected call.
- **Fail-open on hook crash (let the tool run).** Rejected on the merits — a crashing security hook would allow a tool to execute with unredacted inputs, directly leaking sensitive data. It is recorded here as rejected, but it was nonetheless the *shipped* behaviour until PR #253, because it is the framework's default and this ADR assumed otherwise without checking. A rejected alternative is only rejected if something in the code enforces the rejection.
- **Patch or fork the Strands steering handler.** Rejected: it is a vendored dependency on the normal upgrade path; a fork would have to be re-applied on every bump, and the failure mode of forgetting is silent re-opening of the hole. Compensating in our own base class costs ~60 lines and survives upgrades.
- **Let the guard re-raise and rely on agent-level retry → DLQ.** Rejected: this is what the ADR previously claimed happened, and it is not reachable — the framework swallows the exception before any agent-level path sees it. Even if it were reachable, retrying a broken guard repeats an identical failure at full token cost.
- **Post-call placement for WinnerDisclosureGuard.** Rejected: a POST-CALL hook runs after tool execution; if the tool auto-sends before the hook fires, the redaction is too late (corrected v1.0.5).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Fail-closed security actually enforced in code — a broken guard blocks the action rather than leaking | A buggy or crashing guard becomes a hard availability stop for its tool |
| Clean separation: "policy says no" (cheap agent retry with fixed params) from "guard broke" (call cancelled, model told to stop and report — never a silent pass) | The no-retry guidance means a broken guard stops the negotiation at that point rather than degrading around it |
| The invariant lives in one base class every guard inherits, so a new guard is fail-closed by default rather than by author discipline | Guard authors must override `check()`, not `steer_before_tool` — a convention nothing in the framework enforces, so a guard that overrides the wrong method silently opts out |
| A broken guard is now observable — `governance.hook_error` and its alarm distinguish "a guard fired" from "a guard is broken" | One more alarm on the dev bill, and it routes to a topic with no subscribers today (see Results) |

## Results

The fail-closed guarantee is the headline result, and the headline correction: it was stated here from v1.0.5 and only became true on 2026-08-03. What it rests on today is `GuardHandler`, plus disclosure being impossible by construction for `WinnerDisclosureGuard` (deterministic template render, winner fields not parameters) and GUIDE/INTERRUPT cancellation of side-effecting calls before they fire.

`packages/buyer_agent_core/tests/test_steering.py` pins both halves. `test_framework_path_is_fail_open_without_the_base` drives a raw `SteeringHandler` through Strands' own `provide_tool_steering_guidance` and asserts `cancel_tool` is `False` — the upstream behaviour this base class compensates for. Its sibling proves `GuardHandler` sets it. If Strands ever fixes the default, the first test fails and tells us the class can be retired; that is the intended signal, not a broken test.

**Two artifacts this ADR previously claimed as outputs do not exist and were never implemented:** the `steering.hook.rejection_count` metric and its ">3 rejections in 30s" alarm (REQ-A951), and `steering.action` emitted as `PROCEED` / `GUIDE` / `INTERRUPT`. Neither appears anywhere in impl. (`fallback_rejection_count` under `procurement/resilience` is a different signal — Kraljic fallback rejections, AD-100 — and is not a substitute.) What guards actually emit is `governance.violation_count` under `procurement/business`, watched by the `governance_violation` and `governance_confidentiality_violation` alarms; PR #253 adds `governance.hook_error` under the same namespace, dimensioned by guard, watched by `governance_hook_error`. The early-warning-before-DLQ trade-off the old text claimed was therefore never realized in that form.

The availability cost stands as originally accepted: when TCO or risk assessment is persistently unavailable, the `TCOEnforcementGuard` / `RiskAssessmentEnforcement` GUIDE loop is the expected escalation path, not a bug, and it parks the negotiation in REQUIRES_ATTENTION (AD-16) for an operator rather than guessing.

**Open item.** `governance.hook_error` should be zero forever — any datapoint is a code defect, not a business event — but its alarm publishes to `dev-buyer-team-evaluation-alerts`, which has **0 subscribers** (verified live during PR #252). A broken safety guard currently pages nobody. That is the same reasoning that deleted AD-34's evaluation alarms in AD-133, so this alarm is a deletion candidate on its own terms unless a subscription is added. The alarm is also **merged but not applied** — dev was mid-pause when PR #253 landed, and AD-133 requires alarm edits to be applied while dev is up.

Realization was verified live (PRD-003 v1.1.3) on `BidConfidentialityGuard` / `WinnerDisclosureGuard` / `AuctionIntegrityGuard`. The fourth guard named in the previous version of this line, `EvaluationCompletenessGuard`, no longer exists (AD-117).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
