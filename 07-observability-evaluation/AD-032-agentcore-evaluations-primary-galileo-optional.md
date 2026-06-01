# AD-032 — AgentCore Evaluations Primary, Galileo Optional, Three Evaluator Types

**Theme:** Observability & Evaluation  **Catalog:** AD-32 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-33, AD-34, AD-35, AD-49

## Context

Different agent outputs require different kinds of judgement. Tone and strategic soundness are qualitative; classification and ranking accuracy are deterministic; format and policy adherence are structural. Each kind has a different cost and latency profile. Using one evaluation mechanism for all outputs would either overpay (LLM-as-Judge applied to a structural check) or underserve (a code check grading tone). Galileo offers a significantly cheaper per-evaluation rate (~97% cheaper than Bedrock-based LLM-as-Judge) but would introduce an external vendor dependency and trace egress if made the production primary.

## Decision

Adopt AgentCore Evaluations as the production primary, with three evaluator types matched to the question being asked:

- **LLM-as-Judge** (medium cost, seconds) for qualitative metrics — tone, rationale quality, strategic alignment.
- **Ground Truth** (low cost, milliseconds) for deterministic accuracy against golden datasets (Kraljic classification, bid ranking).
- **Code-based / Lambda** (lowest cost, milliseconds) for structural checks — schema, required fields, policy adherence.

Galileo is an optional, feature-flagged addition (`galileo_observability_enabled`, `galileo_protect_enabled` in DynamoDB system-config), recommended for development and staging iteration, with graceful fallback to AgentCore-only if unreachable.

## Alternatives Considered

- **Galileo as production primary.** Rejected: introduces an external vendor dependency and trace egress in production; the ~97% per-eval cost saving does not outweigh the dependency risk at production baseline.
- **LLM-as-Judge for all evaluators.** Rejected: structural and deterministic checks do not require an LLM; using LLM-as-Judge for everything overpays by orders of magnitude and adds latency unnecessarily.
- **Code-based / Lambda for all evaluators.** Rejected: deterministic code cannot grade qualitative output such as tone or rationale quality.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Evaluation cost is contained by construction: structural and ground-truth checks carry most of the load at millisecond latency and lowest cost | LLM-as-Judge evaluators are still medium-cost; their concentration in the qualitative tier directly motivates AD-33's sampling strategy |
| No external vendor dependency or trace egress in production | Teams lose the ~97% per-eval Galileo cost saving in production (retained in dev/staging) |
| `galileo_protect_enabled` hard-defaults to `false` when config is unreachable (AD-49), keeping security posture monotonic under config failure | Galileo evaluation richness (Luna-2 agentic metrics, hallucination detection) is absent from the production baseline |

`galileo_protect_enabled` hard-defaulting to `false` on config unavailability (AD-49) means a config outage leaves Galileo protection disabled rather than preserving a possibly-stale enabled state — a deliberate safety-monotonic design choice.

## Results

Evaluation cost is contained by design, with structural and ground-truth checks carrying most of the load cheaply. Teams get a significantly cheaper iteration loop in non-production environments without exposing production to a third-party dependency. The cost of the qualitative tier (LLM-as-Judge) is concentrated in one evaluator, which directly motivates AD-33's tiered sampling for Communication Tone. AD-34 (closed-loop automated actions) consumes the scores produced by this evaluator set, and AD-35 extends coverage with scheduled ATLAS-specific evaluators that run outside the per-trace online path.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
