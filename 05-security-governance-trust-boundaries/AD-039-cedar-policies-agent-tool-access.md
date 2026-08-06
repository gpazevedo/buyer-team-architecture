# AD-039 — Cedar Policies for Agent→Tool Access; the §5.1 Table Is Authoritative

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-39 · **Source PRD:** PRD-005 · **Status:** Accepted — narrowed 2026-08-06: Cedar governs PO Receiving only; see the correction below for why it was never applicable to the 6 LLM agents · **Related:** AD-38, AD-40, AD-41, AD-36, AD-135

> **Correction (2026-08-06) — this ADR's Decision was never going to be buildable for the 6 LLM agents, on any Gateway in the system.** The Decision below states Cedar is used "at the AgentCore Gateway boundary" to authorize the ~60 agent→tool rules the §5.1 table specifies. Tracing the actual call path found that these agents' domain tools (`calculate_tco`, `assess_supplier_risk`, and the rest) are Strands-native, in-process `@tool`s — `buyer_agent_core` re-exports `strands.tool` directly (AD-135) — so a tool call never leaves the agent's own process. There is no network hop for a Gateway-scoped Cedar engine to evaluate against, regardless of how the `.cedar` rules are written or how long provisioning is deferred. Earlier framing described this as "not yet provisioned," which implied a pending deployment; it was never one.
>
> Two other Gateways were checked before settling on this. The ingest Gateway (`infra/gateway.tf`) — previously the implied eventual attachment point — fronts a different tool (`ingest_purchase_requisitions`), not any of the 6 agents' own tools, and isn't fronted by Cedar at all: Cognito JWT auth plus the tenant-injecting Interceptor (AD-41) only. The one Cedar deployment that exists anywhere in `infra/policies/` is `po_receiving.cedar`, on a third Gateway (PO Receiving) with a single M2M principal, and it is one coarse `permit(principal, action, resource==gateway)` — not the ~60 fine-grained rules this ADR specifies — because an AgentCore Runtime (HTTP) target exposes exactly one action to Cedar in the first place. No rule count written against the §5.1 table would have been expressible on that target type either.
>
> The §5.1 table is not struck from the record — it still documents real intent, which agent should be able to reach which tool — but it is corrected below from "the source Cedar policy files are generated from" to what it actually is: a permission matrix with no enforcement mechanism behind it. The control that actually occupies Layer 3's position for the 6 agents' own tool calls is the Strands steering-hook layer (AD-23, AD-24) — its fail-open failure mode, not Cedar provisioning, was the real priority gap, and PR #253/#262 have since closed most of it.

## Context

Which agent may call which tool must be enforced outside agent code, so prompt injection cannot grant new tool access. There must also be one unambiguous source for those permissions to avoid drift between narrative documentation and deployed policy. Without an authoritative source, doc/policy drift can silently produce either over-permissive or under-permissive rules.

## Decision

**As originally decided (superseded for the 6 agents — see the correction above):** use Cedar at the AgentCore Gateway boundary with default-deny and forbid-overrides-permit, generating ~60 rules across 6 agents from the PRD-005 §5.1 permission table.

**What actually governs agent→tool access today.** The §5.1 table remains the authoritative statement of *intent* — in any conflict between narrative documentation (PRD-003 §5) and this table, the table prevails — but Cedar is not, and structurally cannot be, its enforcement mechanism for the 6 LLM agents. Their tools are in-process function calls; there is no Gateway hop for Cedar to intercept. Enforcement instead sits at two other points: a tool simply not being registered in an agent's own toolset (the mechanism actually observed for e.g. Spot Bidding → `generate_award_notification`, PRD-005 §11.1), and the steering-hook layer (AD-23/AD-24) for the business-logic invariants a binary permit/forbid rule could never express anyway (confidentiality, budget ceilings, disclosure).

## Alternatives Considered

- **Prompt-stated tool restrictions.** Rejected: bypassable by prompt injection; the model cannot be relied upon to enforce its own access boundaries.
- **IAM-only tool access control.** Rejected: IAM operates at the AWS API level and cannot enforce the fine-grained per-agent, per-tool semantics required by the §5.1 matrix — moot for the 6 agents regardless, since there is no AWS API boundary for an in-process tool call either.
- **Wrap the 6 agents' tools behind a Gateway just to give Cedar something to authorize.** Never proposed in any PR or design doc, and not evaluated as a real alternative here either: it would add a network hop and a coarse, one-action authorization boundary (per the PO Receiving precedent below) purely to satisfy a mechanism, not to close a threat the steering-hook layer doesn't already cover.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Cedar remains real, deployed enforcement for PO Receiving (external partner traffic, the one Gateway-fronted surface in this system) | The §5.1 table's ~60 rules describe intent with no enforcement mechanism behind them for the 6 LLM agents — a permission matrix, not a policy |
| The steering-hook layer (AD-23/AD-24) already covers the business-logic invariants a fine-grained Cedar rule set would have targeted, so no capability gap opened when this correction landed | Tool-registration omission (not a Cedar DENY) is what actually stops an agent from reaching an unintended tool — a weaker, code-review-dependent boundary than a policy engine |

One such drift once caused Cedar deny-by-default to wrongly block the permitted `generate_award_recommendation` tool on the design this ADR originally specified — a Layer-3 authorization fault, not a Layer-5 steering-hook failure — fixed in v1.2.0. That incident predates this correction and describes the §5.1 table's role as a *doc* source of truth (PRD-003 §5 is explicitly non-authoritative against it); it does not imply Cedar was ever the enforcement layer reading that table for these agents.

## Results

Cedar governs exactly one Gateway in this system: PO Receiving (`po_receiving.cedar`, LOG_ONLY, one M2M principal, one coarse permit — AD-40 governs its rollout). It does not, and cannot as currently architected, govern the 6 LLM agents' own tool calls, because those calls are in-process Strands `@tool` invocations with no Gateway in the path (AD-135). The PRD-005 §5.1 table is retained as a permission-intent record, explicitly no longer described as "generated into Cedar policy files." Cedar rollout phases (AD-40) apply only to PO Receiving. Cedar remains Layer 3 in the six-layer stack (AD-36) in name; in practice, for the 6 agents, Layer 3's role is filled by tool registration plus the steering-hook layer (AD-23/AD-24), and by Bedrock Guardrails (AD-43) for content filtering at Layer 4.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
