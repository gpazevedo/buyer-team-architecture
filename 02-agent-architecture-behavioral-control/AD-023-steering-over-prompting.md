# AD-023 — Steering over Prompting

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-23 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-22, AD-24, AD-28

## Context

Behavioral guardrails must hold every time — never leak competitor pricing to a supplier (REQ-A200), always calculate TCO before recommending an award (REQ-A400), redact the winner from rejection letters (REQ-A600). These can be written as prompt instructions, which the model follows probabilistically, or enforced as runtime hooks that intercept tool calls. PRD-003 §1.1(3) cites measured compliance: prompt instructions at 82.5%, runtime hooks at 100%. Prompt-based guardrails can also be bypassed by adversarial supplier content injected into bid text.

## Decision

Behavioral guardrails are implemented as Strands Steering Hooks that intercept tool calls at runtime, not as prompt text. Because hooks operate outside the agent's reasoning loop, the LLM cannot forget, be talked out of, or be injected past them.

## Alternatives Considered

- **Prompt-only guardrails.** Rejected: measured compliance of 82.5% means roughly one-in-six violations; prompt text is also vulnerable to prompt-injection from adversarial supplier content.
- **Post-processing output filters.** Rejected: a filter applied after the tool fires cannot prevent a side effect that has already occurred (e.g., a bid invitation sent with competitor pricing embedded); enforcement must be pre-call for side-effecting tools.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Deterministic enforcement — 100% vs 82.5%; the gap justifies the approach | Hooks are code that must be written, deployed, and version-matched to the tools they guard |
| Resistance to prompt injection — a guardrail in a hook cannot be overridden by adversarial supplier content in bid text | Added pre-call latency and a new failure mode: a hook can wrongly reject valid work |
| Independently unit-testable as pure Python functions | Rejection handling becomes its own subsystem — the 5-rejection escalation ladder and early alarm (REQ-A951) exist because of this |
| Keeps governance out of the prompt, reinforcing cache-prefix purity (AD-28) | A misconfigured hook can block forward progress entirely |

## Results

The 82.5 → 100% compliance delta is the core result — guardrails that previously failed roughly one time in six now hold every time. This decision depends on AD-22 (tools as boundaries) to have a call-interception point. It drives the design of AD-24 (the specific failure semantics chosen for hook crashes and rejections), including the rejection-loop machinery, the `steering.hook.rejection_count` alarm, and the `excluded_bid_count` correction — all downstream consequences of moving enforcement into code rather than prose.

**Realization note (PRD-003 v1.1.3).** The "steering over prompting" decision stands; only the hook *mechanism* was reconciled to the shipped Strands 1.43 API — all guards now run PRE-CALL in **GUIDE** mode (cancel + corrective guidance), since that API exposes no tool-I/O mutation and no after-tool hook. The "post-processing output filters" alternative rejected above is in fact reinforced by this: the shipped API has no after-tool stage, so pre-call enforcement is the only option for side-effecting tools. See AD-24 for the revised semantics.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
