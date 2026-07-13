# AD-022 — Tools as Boundaries

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-22 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-23, AD-24, AD-26, AD-28, AD-39

## Context

Every agent needs to take external actions — send a bid invitation, score a bid, calculate TCO, query supplier history. Those actions and their data could be expressed as free text inside the system prompt, or every external action could be forced through a typed tool call. The choice has security, caching, and auditability consequences well beyond convenience: embedding data in the system prompt invalidates the prompt-cache prefix on every invocation, and a prompt-embedded action cannot be intercepted by a Cedar policy or a steering hook.

## Decision

Every external action is a tool call, never embedded in the system prompt. The system prompt carries reasoning instructions only; data in and side effects out flow exclusively through tools. Each state-mutating tool additionally carries a dedup key and idempotency contract.

## Alternatives Considered

- **Prompt-embedded actions (free-text instructions in system prompt).** Rejected: prompt text cannot be intercepted by Cedar or steering hooks, invalidates the prompt-cache prefix on every invocation, and produces no structured audit record per action.
- **Hybrid — read tools, prompt-embedded writes.** Rejected: write actions are exactly the ones that need Cedar interception (AD-39) and steering-hook guardrails (AD-24); exempting writes from the tool boundary removes the enforcement point where it matters most.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Security interception point — Cedar (Layer 3, AD-39) and steering hooks (Layer 5, AD-24) sit at the tool boundary, so prompt injection cannot directly fire a side effect | Every action needs a designed tool contract, schema, and dedup key — more engineering per capability |
| Cache-prefix stability — keeping data out of the prompt keeps the `system` block invocation-invariant, enabling ~90% input-token savings (AD-28) | Agents cannot "just say" something; even a clarification is a `send_clarification_request` call |
| Full auditability — every action emits an OTEL span with `procurement.tool.name`, `negotiation_id`, `supplier_id` | A wider tested surface: each tool needs failure-mode and idempotency tests |

## Results

Tools-as-Boundaries is the structural precondition for three downstream decisions: it makes steering hooks possible (AD-23 and AD-24, which intercept tool calls), it makes prompt-cache purity achievable (AD-28), and it gives idempotency a place to live (AD-26). The Cedar per-agent permit/forbid table (AD-39) is only enforceable because every external action surfaces as a typed tool call at the Gateway boundary. The residual cost is steady — every new agent capability is a tool with a contract, not a prompt sentence.

**Update 2026-07-12 (PR #201): two agents' tool surface narrowed by moving pure-lookup work out of the loop entirely.** `bottleneck_negotiation_llm` and `strategic_partnership_llm` no longer carry `assess_supplier_risk`/`calculate_tco` (bottleneck) or the equivalent relationship-history/risk/TCO lookups (strategic partnership) as tools — `orchestrator/node_strategy_execute.py` now precomputes `risk_assessments`/`tco_results`/`relationship_histories` for every candidate supplier before invoking the agent, passing them in on the request. The agents' remaining tool surface narrows to the side-effecting calls only: a single batched `send_negotiation_proposals` (replacing a prior one-call-per-supplier `send_negotiation_proposal`) and `generate_award_recommendation`. Steering-hook guards (`TCOEnforcementGuard`/`RiskAssessmentEnforcement`) were rewritten to match — they no longer count prior tool-call history (`assess_supplier_risk`/`calculate_tco` calls no longer happen to count), instead checking that every `candidate_suppliers` id has a matching entry in the precomputed `tco_results`/`risk_assessments` arguments passed to `generate_award_recommendation` itself. This is a narrower guarantee than before (argument-list completeness, not calculation provenance — nothing stops the model from fabricating an entry instead of relaying the precomputed one through), an explicitly accepted trade-off of moving deterministic work out of the model's tool-call loop. The Decision's core principle is unaffected: every remaining side-effecting action for these two agents is still a typed, Cedar-interceptable, steering-hook-guarded tool call — only the boundary moved, narrowing to exclude pure lookups that don't need an LLM turn. `leverage_auction_llm` and `spot_bidding_llm` are untouched by this change.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
