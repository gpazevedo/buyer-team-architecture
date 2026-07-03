# AD-043 — Bedrock Guardrails on All Agents

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-43 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-36, AD-23, AD-39

## Context

Supplier bids and emails are untrusted input that may carry prompt-injection payloads, PII, or attempts to steer the agent off-topic (OWASP ASI01 goal hijacking, exfiltration, RAG credential harvesting). This content must be filtered before the agent reasons over it. Without a content filter at the model boundary, behavioral steering hooks (Layer 5) and Cedar (Layer 3) are the first line of defense against injection — a weaker posture than filtering at ingestion.

## Decision

Apply Amazon Bedrock Guardrails — PII detection, topic restriction, prompt-injection detection at PROMPT_ATTACK:HIGH, and credential-pattern regex — to all agents as Layer 4 content filtering in the six-layer stack (AD-36). Guardrails is applied on every input and output, not sampled.

## Alternatives Considered

- **Bespoke input-validation code per agent.** Rejected: high implementation cost, inconsistent coverage across agents, and no managed updates as attack patterns evolve.
- **Rely solely on behavioral steering hooks (Layer 5).** Rejected: hooks enforce behavioral policy but do not filter content; injection payloads that reach the reasoning loop can still influence output even when hooks block specific tool calls.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Managed, model-level filter on every input/output covering several OWASP/ATLAS techniques without bespoke code per agent | Guardrails is an LLM-based control and can degrade or be probed; the ATLAS table rates reconnaissance/staging coverage only "Partial" |
| Consistent Layer 4 coverage across all 7 agents from a single configuration | It blocks the terminal injection step but does not detect the offline staging phase; it cannot be the sole defense |

Resilience fallbacks must never bypass Cedar or partition-key isolation; Guardrails degradation under fallback is acceptable precisely because Layers 1–3 remain. Guardrail triggers emit a span attribute feeding an anomaly alarm (REQ-S601).

## Results

Bedrock Guardrails is combined with `<supplier_data>` delimiter wrapping and behavioral steering hooks (Layer 5, AD-23) to create overlapping injection resistance. Guardrail trigger events emit span attributes that feed the anomaly alarm (REQ-S601). As an LLM-based control, Guardrails is never the sole defense — it is Layer 4 in the AD-36 stack, sitting above Cedar (Layer 3, AD-39) and below behavioral hooks (Layer 5, AD-23).

**Not wired, as of the AD-035 live replay, 2026-07-03.** The Guardrail resource itself is provisioned (`infra/modules/security/main.tf`), but no agent's Bedrock invocation actually applies it: a codebase-wide search finds zero `guardrailConfig` usage anywhere in `impl/` application code (only in the `strands`/`litellm` SDK internals, which expose the parameter but are never passed one). This was surfaced, not assumed — AD-035's adversarial robustness evaluator replayed 60 attack prompts against the live `dev_kraljic_classifier` runtime and scored 0/60 blocked; the raw response for each is a generic `"Expecting value: line 1 column 1 (char 0)"` JSON-parse failure, not the Guardrail's literal block message, confirming the model boundary has no filter attached. Layers 1–3 (Cedar, partition-key isolation) and Layer 5 (behavioral steering hooks, AD-23) are unaffected by this gap and remain the active defenses today. Wiring `guardrailConfig` into each agent's model client call is the remaining work to close this ADR.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
