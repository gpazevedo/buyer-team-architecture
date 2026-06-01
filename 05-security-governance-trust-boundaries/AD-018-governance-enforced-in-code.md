# AD-018 — Governance Enforced in Code at the Node Level

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-18 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-23, AD-11, AD-19, AD-39

## Context

Governance policies — spend-approval thresholds, max negotiation rounds, ESG minimums, approved-supplier filters, communication approval, budget ceilings, classification override — must be enforceable in a way that cannot be subverted. Anything expressed only as prompt instruction can be ignored by the model or defeated by prompt injection. The graph nodes provide deterministic, auditable enforcement points outside the LLM's control.

## Decision

Enforce every governance policy in deterministic code at a specific node, never as prompt text. Spend approval and the quality/amount gates live in Node 6; max rounds and budget ceiling are passed as hard constraints into Node 4x; the approved-supplier filter runs at Node 1; ESG is enforced two-tier (a binary candidate-pool gate at Node 3 plus weighted 10–15% scoring at Node 5); communication approval is the `auto_send_communications` flag at Node 7; classification override is gated at Node 2. Every governance decision is captured as an OTEL span attribute for audit.

## Alternatives Considered

- **Governance via system-prompt instructions.** Rejected: bypassable by injection, non-auditable, and the model can simply not comply.
- **Governance entirely outside the graph (e.g. a downstream review service).** Rejected: loses the ability to halt or branch at the decision point; turns prevention into after-the-fact detection.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Tamper-resistant enforcement — policies cannot be bypassed by prompt injection because enforcement is outside the LLM's control | Policy fluidity — new governance mechanisms require a node-level code change, not a prompt edit |
| Full audit trail — every governance decision is captured as an OTEL span attribute | Threshold and mapping changes flow through `{env}-system-config` without redeploy, but structurally new policies still require deliberate graph changes |

The ESG two-tier design is a concrete instance of the added structure — one binary gate plus one weighted score — rather than a single prompt rule. Most tuning (thresholds, mappings) is mitigated by sourcing from `{env}-system-config`.

## Results

Governance cannot be bypassed by prompt injection because the enforcement points are outside the LLM's control. Every governance decision is in the distributed trace, satisfying auditability requirements. Threshold and mapping changes flow through config without redeploy; structural policy changes are deliberate, reviewed graph changes. Cross-references: AD-19 handles the user-action authorization plane at the same interrupt-resume API; AD-23 (steering hooks) operates at Layer 5 within the same defense-in-depth stack (AD-36).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
