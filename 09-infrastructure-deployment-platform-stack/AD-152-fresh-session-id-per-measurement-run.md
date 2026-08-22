# AD-152 — A Measurement Run Mints Fresh Session IDs; Only Production Reuses Them

**Theme:** Infrastructure, Deployment & Platform Stack
**Catalog:** AD-152 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-34, AD-56, AD-54, AD-90, AD-103, AD-105

## Context

AgentCore Runtime pins a `runtimeSessionId` to a warm microVM, and **that microVM keeps serving the agent image it booted with even after the endpoint has rolled to a new version**. `get_agent_runtime_endpoint` reports the new version as `READY` throughout — the endpoint really has rolled; what has not rolled is the warm microVM still bound to an existing session id. Any caller that derives its session id from stable inputs therefore addresses the *pre-deploy* image for as long as that session's TTL lasts, with no error, no warning, and a green deployment.

This is normally a feature. Warm-session reuse is why `node_kraljic_classify.py`'s deterministic `kraljic-{tenant_id}-{category_id}` exists — it is what makes the agent's `load_axes` `lru_cache` pay off (AGENTCOST06-BP03), and the same reuse underpins the warm-path assumptions in AD-105 and the session affinity the skill transports depend on.

It stops being a feature the moment the caller's purpose is *measurement*. `orchestrator/eval_kraljic_accuracy.py`'s `predict_agent` invoked the agent with `session_id=f"eval-{category_id}"` — identical on every run, by construction. So the AD-034 accuracy gate, whose entire premise (REQ-O222) is that a change in live accuracy is attributable to a spec revision, was free to score whichever image the warm microVMs happened to hold. On 2026-08-22 it did exactly that: the post-PR-#354 run scored `name_only` 16/20 = 80.0% with four misses on the profit axis, contradicting the change it was supposedly measuring. `spend_materiality_index` — the field PR #354 added — appeared **0 times in 30 days** of the runtime's log group, while the run's tool outputs still carried `spend_trend`, the field PR #354 deleted. The deployed image was correct (`tools.py` extracted from the ECR layer behind v102 had both new functions; the DEFAULT endpoint went `READY` on v102 twenty-one minutes before the run). Reproduced directly, same endpoint, 90 seconds apart: 20 fresh `uuid4` sessions all emitted `spend_materiality_index`; one reused `eval-<category_id>` session still emitted `spend_trend`.

## Decision

Split the session-id convention by intent, and make the split explicit at both call sites.

- **Anything that measures a deployed image mints a fresh session id per run.** The Kraljic gate now derives its ids from a per-run `RUN_ID` (`eval-{RUN_ID}-{category_id}`), so no run can ever land on a microVM warmed by a previous one.
- **A run id that scopes invocation must also scope read-back.** `evals/agentcore/builtin_kraljic_eval.py` reads its OTEL spans back *by* session id, so a bare prefix change would have silently stopped it matching anything; it takes a `--run-id` argument to scope its lookups to one harness run. The two halves are kept in sync by that shared id, not by convention.
- **Production keeps deterministic session ids on purpose.** `node_kraljic_classify.py` continues to use `kraljic-{tenant_id}-{category_id}`. The identical one-session-TTL staleness applies there, knowingly: a production caller wants the warm cache, and a deploy's tail of stale-image invocations bounded by a session TTL is an accepted cost of the canary model (AD-56), not a defect to fix here.

## Alternatives Considered

- **Wait out the session TTL before running a gate.** Rejected: turns a correctness property into a timing convention nobody can verify was followed, and the TTL is not a documented, controllable number.
- **Assert the image digest before scoring (query the endpoint version and compare).** Rejected as insufficient rather than wrong — the endpoint reported the correct new version throughout this incident. Version reporting is exactly the signal that failed to discriminate.
- **Make every caller mint fresh session ids.** Rejected: it would discard warm-microVM reuse in production, where AD-105's warmup and the `load_axes` cache depend on it, to fix a problem that only exists for measurement.
- **Status quo / stable ids everywhere.** Rejected: it makes any gate run inside a session TTL of a deploy silently attribute the old image's behaviour to the new spec, which is the precise thing AD-34's closed loop acts on.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A gate result is attributable to the image that was deployed when it ran — REQ-O222's premise becomes true rather than assumed | Every gate row now pays a cold start; the harness is slower and marginally more expensive per run |
| The failure mode is eliminated by construction, not by a check that has to be remembered | Two conventions now exist for one platform primitive, and picking the wrong one is silent in both directions |
| Read-back stays aligned with invocation through one explicit `--run-id` | Any future consumer of these spans must learn that session ids are per-run, or it will match nothing |

The asymmetry is the point: **reusing a session id costs correctness only when the thing being measured is the deployment itself.** Everywhere else it is the cheap path, which is why the general rule is not "never reuse" but "never reuse in a measurement run."

## Results

Shipped in impl PR #356 (`orchestrator/eval_kraljic_accuracy.py`, `evals/agentcore/builtin_kraljic_eval.py`). Re-scored through the patched harness against the same deployed endpoint, `name_only` went 16/20 = 80.0% → **18/20 = 90.0%**, with all four profit-axis misses (Catering, Professional Services, Security Services, R&D) resolving — so the 80.0% that would otherwise have been recorded as [AD-005](../12-procurement-domain-logic/AD-005-kraljic-2x2-routing-primitive.md)'s PR #354 result, a bare pass exactly at the 0.80 floor, was a measurement of the image that PR replaced. The two residual misses are genuine boundary cases on the supply axis, not this failure mode.

Consequences for anything downstream of a deploy: the AD-034 closed loop's `kraljic_accuracy` threshold now reads a number about the current image; the canary model (AD-56) keeps its warm-session tail deliberately; and the production deterministic-id path in Node 2 is left as a known, accepted staleness window awaiting its own decision if it ever needs one. The general form of this hazard — a deployed-version report that stays green while a warm session serves the old code — is not specific to evaluation and should be assumed to apply to any post-deploy verification that talks to an AgentCore Runtime.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
