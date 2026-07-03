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

**Two gaps found 2026-07-03, both closed in code and merged, neither yet applied to dev.** AD-035's adversarial robustness evaluator replayed 60 attack prompts against the live `dev_kraljic_classifier` runtime and scored 0/60 blocked; the raw response for each was a generic JSON-parse failure, not the Guardrail's literal block message. That surfaced two independent problems, both fixed in `buyer-team-impl` PR #120 (merged to `main`):

- **Wiring.** No agent's Bedrock invocation ever applied the Guardrail resource — a codebase-wide search found zero `guardrailConfig` usage anywhere in `impl/`. `DynamicAgentFactory.build()` — the single `BedrockModel(...)` construction site in the whole repo — now passes `guardrail_id`/`guardrail_version` via `buyer_agent_core/guardrail.py`, sourced from `GUARDRAIL_ID`/`GUARDRAIL_VERSION` env vars that `infra/agent_runtimes.tf` sets on all 7 agent runtimes from `module.security`'s outputs, plus the `bedrock:ApplyGuardrail` IAM permission the runtime roles were missing.
- **Policy coverage.** The Guardrail resource itself only ever configured `content_policy_config`'s `SEXUAL`/`VIOLENCE`/`HATE`/`INSULTS` filters — none of the four controls this ADR's Decision promises (PII detection, topic restriction, `PROMPT_ATTACK:HIGH`, credential regex) existed, so even a correctly-wired call would not have tripped it against AD-035's dataset. `aws_bedrock_guardrail.buyer_team` now also configures a `PROMPT_ATTACK` content filter, `sensitive_information_policy_config` (PII entities including native AWS credential detection, plus a supplementary regex for other password/token assignments), and `topic_policy_config` denying instruction-override/governance-bypass requests.

Both changes are unit- and integration-tested (`terraform validate`/`tflint`/`trivy`/`checkov` clean) but **not yet applied to dev and not re-verified against a live invoke** — AD-035 should be re-run after a `terraform apply` before this ADR's Results can claim the control is actually effective in the deployed system, not just present in code.

Layers 1–3 (Cedar, partition-key isolation) and Layer 5 (behavioral steering hooks, AD-23) are unaffected by this gap and remain the active defenses today.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
