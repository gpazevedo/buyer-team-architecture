# AD-039 — Cedar Policies for Agent→Tool Access; the §5.1 Table Is Authoritative

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-39 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-38, AD-40, AD-41, AD-36

## Context

Which agent may call which tool must be enforced outside agent code, so prompt injection cannot grant new tool access. There must also be one unambiguous source for those permissions to avoid drift between narrative documentation and deployed policy. Without an authoritative source, doc/policy drift can silently produce either over-permissive or under-permissive rules.

## Decision

Use Cedar at the AgentCore Gateway boundary with default-deny and forbid-overrides-permit. The per-agent permission table in PRD-005 §5.1 is the authoritative source from which Cedar policy files are generated; in any conflict between the narrative documentation and the deployed policy, the §5.1 table prevails. Approximately 60 rules across 6 agents are generated from this table.

## Alternatives Considered

- **Prompt-stated tool restrictions.** Rejected: bypassable by prompt injection; the model cannot be relied upon to enforce its own access boundaries.
- **IAM-only tool access control.** Rejected: IAM operates at the AWS API level and cannot enforce the fine-grained per-agent, per-tool semantics required by the §5.1 matrix.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Enforcement external to agent reasoning — prompt injection cannot grant new tool access | Cedar authorizes the tool binding but cannot inspect the request body; it must be paired with the Interceptor (AD-41) and predicate rewriting (AD-70) for tenant-correctness within a permitted call |
| Single generated-from source prevents doc/policy drift between the narrative table and deployed policy | ~60 rules across 6 agents must be maintained in sync with the §5.1 table as the permission matrix evolves |

One such drift once caused Cedar deny-by-default to wrongly block the permitted `generate_award_recommendation` tool — a Layer-3 authorization fault, not a Layer-5 steering-hook failure — fixed in v1.2.0. The generated-from-source approach is the structural fix. The narrative summary in PRD-003 §5 is explicitly non-authoritative.

## Results

~60 Cedar rules across 6 agents are generated from the PRD-005 §5.1 table. The narrative summary in PRD-003 §5 is explicitly non-authoritative. Cedar rollout phases are governed by AD-40. Cedar is Layer 3 in the six-layer stack (AD-36); it is coarse-grained and must be paired with the Gateway Interceptor (AD-41) for tenant-correct authorization within permitted calls, and with Bedrock Guardrails (AD-43) for content filtering at Layer 4.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
