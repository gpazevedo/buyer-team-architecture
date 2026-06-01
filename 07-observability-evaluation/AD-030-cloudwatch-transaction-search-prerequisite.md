# AD-030 — CloudWatch Transaction Search as a Hard Prerequisite for Evaluations

**Theme:** Observability & Evaluation  **Catalog:** AD-30 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-29, AD-31, AD-32, AD-34, AD-8

## Context

AgentCore Evaluations score agent sessions by reading their traces. Those traces are reachable by the evaluators only if CloudWatch Transaction Search is enabled on the account — a one-time setup step. If it is off, both on-demand and online evaluations have no input and silently cannot score sessions; the full set of platform metrics also fails to appear. The consequence is that the entire closed-loop quality system (AD-34) degrades invisibly with no loud failure signal.

## Decision

Treat CloudWatch Transaction Search as a documented hard prerequisite of the observability and evaluation architecture rather than building an alternative trace-access path. Enable it once per account as part of standing up the environment, validated at environment provisioning time.

## Alternatives Considered

- **Custom trace-export pipeline to feed evaluators.** Rejected: duplicates an AWS-native capability, adds engineering and operational surface, and introduces a bespoke pipeline that must be maintained independently across environments.
- **Status quo / no action.** Rejected: without Transaction Search, on-demand and online evaluations silently produce no scores, disabling quality control entirely (AD-34) and making the system unobservable to evaluators.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Evaluations and full platform metrics work without further wiring once the toggle is set | A missing one-time account-level toggle silently disables the entire closed-loop quality system (AD-34) with no automatic detection |
| No bespoke pipeline to maintain; relies on the managed AWS-native path | The toggle is easy to miss during environment setup; its absence surfaces only as "evals aren't scoring" |
| Reinforces the system-wide region constraint (AD-8) that the system deploy only where AgentCore Evaluations is GA | Deployment is restricted to regions where both Evaluations and Transaction Search are available |

## Results

Once enabled, evaluations and full platform metrics function without further wiring. The architecture gains a single, well-identified preflight check that must be validated at environment provisioning. If skipped, quality monitoring degrades invisibly — the sharpest downside of this decision. AD-32 (evaluation types and Galileo) and AD-34 (closed-loop automated actions) both depend on this prerequisite being satisfied; neither can function correctly if Transaction Search is absent.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
