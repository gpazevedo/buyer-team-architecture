# AD-036 — Six-Layer Defense-in-Depth

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-36 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-38, AD-39, AD-43, AD-18

## Context

An agentic system processing untrusted supplier content and making automated financial decisions has many distinct attack surfaces: prompt injection, tool misuse, cross-tenant access, memory poisoning, and cost harvesting. No single control covers all of them, and any single control can fail. Without an explicit layered model, coverage gaps are invisible and a single failure becomes a breach.

## Decision

Organize security as six independent layers — Infrastructure, Identity, Tool/Entity Access, Content Filtering, Behavioral Guardrails, Application — presented explicitly as a conceptual stack, not a call sequence. The numbering reflects the conceptual model, not execution order; Content Filtering (Layer 4) runs on input before the agent reasons, Cedar (Layer 3) runs per tool call. The stated invariant is that no single layer's failure produces a breach, because another layer still stands between the attacker and the asset.

## Alternatives Considered

- **Single comprehensive security layer.** Rejected: any one control can fail or be bypassed; a single layer provides no residual protection after compromise.
- **Ad-hoc per-feature controls without a unifying model.** Rejected: coverage gaps are invisible and ownership is unclear, making audit and incident response unreliable.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Independence — a single-layer failure is not a breach; another layer remains between the attacker and the asset | Simplicity — six layers are more to build, operate, and reason about than a single control |
| Clear ownership — each concern maps to its own section with explicit OWASP/ATLAS coverage ratings | Redundancy cost — the same check (tenant scoping) recurs across several layers by design |

The layers are a stack, not an order. A reader who assumes call order would misunderstand the flow; the PRD is explicit that numbering reflects conceptual organization. During resilience fallbacks, LLM-based controls (Layers 4–5) may degrade, but infrastructure controls (Layers 1–3) are documented to remain — the fallback table makes that guarantee auditable.

## Results

Each layer is specified in its own PRD-005 section and mapped to OWASP Agentic and MITRE ATLAS techniques, with explicit coverage ratings. The six-layer model is the named principle under which AD-38 (ABAC supplements partition-key checks, Cedar, and predicate rewriting) is articulated. AD-39 (Cedar) is Layer 3; AD-43 (Bedrock Guardrails) is Layer 4; AD-23 (steering hooks) is Layer 5; AD-18 (node-level governance code) operates at the Application layer. Resilience fallbacks document which layers degrade and which remain, satisfying the auditability requirement.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
