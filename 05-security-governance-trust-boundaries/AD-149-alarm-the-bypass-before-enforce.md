# AD-149 — A Degrading Caller Inverts an Enforcement Flip: Alarm the Bypass Before ENFORCE

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-149 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-39, AD-40, AD-41, AD-46, AD-133, AD-147, AD-148

## Context

Five Cedar policy engines now front Gateways in this system (AD-39's 2026-08-19 footprint note), all
running `LOG_ONLY`. Each is expected to reach `ENFORCE` in production eventually, on the three-phase
rollout AD-40 defines. Independently, AD-46 establishes that non-essential dependency failures degrade
rather than halt: the PR→PO handoff must survive a Gateway or Cognito outage, so `orchestrator/po_delivery.py`
catches any gateway exception and writes the PO in-process with boto3.

Both decisions are sound alone. Together they produce a failure mode neither anticipates.

`po_receiving.cedar` is deny-by-default with a single permit conditioned on `principal.getTag("scope")`
resolving. If that tag fails to resolve in an environment — a per-environment M2M client that was never
minted with the `by-app-client` binding rows, a scope named differently, the tag-resolution behaviour
AD-147 had to probe for in the first place — then under `ENFORCE` Cedar denies **every** call. Node 7
does not fail. It catches the 403 and takes the fallback, so purchase orders keep flowing while each one
bypasses the Cedar engine, the REQUEST interceptor (AD-41) and the JWT authorizer in a single step.

The system is then strictly less protected than it was in `LOG_ONLY`, where the same calls passed through
the authorizer and interceptor and were merely logged by Cedar. Nothing external distinguishes this state
from a healthy one: POs are created, the workflow completes, `receiving_policy_mode` reads `ENFORCE` in
the configuration, and the only trace was a `log.warning`. An enforcement flip intended to tighten a
boundary can therefore remove it, and announce success while doing so.

This is not specific to receiving. It is the general shape of any control placed in front of a caller
that degrades gracefully — which, per AD-46, is the house style for every non-essential dependency.

## Decision

**A control may not be moved to `ENFORCE` until the bypass path it can trigger is independently alarmed,
at the caller.** Concretely, for each of the five policy engines: before the flip, the calling code must
classify its own failure-to-reach-the-Gateway cases, emit a metric distinguishing an authorization
rejection from a transport or configuration failure, and carry an alarm on the authorization bucket at
the severity of a bypassed control — not at the severity of a degraded dependency.

Realized for receiving as `procurement/resilience po_delivery.gateway_fallback`, dimensioned by `reason`
(`auth` for 401/403, plus `http`, `config`, `transport`), with the alarm
`${env}-buyer-team-po-delivery-gateway-auth-bypass` on the `auth` bucket routed to `alerts_critical` —
the same tier as `governance_confidentiality_violation`, because both mean a control was bypassed rather
than a service was slow. `transport` and `config` are deliberately unalarmed: transport is transient and
noisy, and alarms bill per alarm-month regardless of state (AD-133).

**The instrumentation must live at the degrading caller, not at the control.** A control cannot observe
its own circumvention. This is the load-bearing half of the decision and the part most likely to be
argued away, so the reasoning is recorded under Alternatives.

For engines whose callers do not yet exist, this condition composes with AD-148's: a real caller **and**
an alarm on that caller's bypass path, both, before any flip.

## Alternatives Considered

- **Alarm on the Gateway's own `DenyDecisions` metric.** Rejected as insufficient, though it is the
  obvious answer and the metric is genuinely there: `AllowDecisions`/`DenyDecisions` publish to
  `AWS/Bedrock-AgentCore` by default, dimensioned by Policy / PolicyEngine / Mode, which is what
  `scripts/manage_gateway_observability.py` exists to enable. Three gaps. It sees the denial, not the
  write that follows it — the boto3 fallback is invisible to the Gateway, so the metric cannot answer the
  only question that matters, whether an unauthorized PO was created. Its meaning inverts across the flip:
  under `LOG_ONLY`, denies are the expected output of the observation phase, so an alarm on them is noise;
  under `ENFORCE` the same datapoint is an incident. And it is silent in exactly the cases that matter
  most — a wrong Gateway URL, an expired client secret, or an unreachable endpoint never produces a Cedar
  decision at all, so `DenyDecisions` stays flat at zero while every PO takes the fallback.
- **Make the fallback fail closed under `ENFORCE`.** Rejected: it converts a Gateway or Cognito outage
  into a halted PR→PO handoff, which is precisely the trade AD-46 decided the other way. The fallback is
  correct; its invisibility was the defect.
- **Flip to `ENFORCE` and rely on the observation window plus manual telemetry review.** Rejected: this
  is what the workflow already claimed to do, and the review it required could not happen — see AD-40's
  2026-08-19 update, where the only step that creates production's engine was the one passing `ENFORCE`.
  A review gate whose input is produced by the action it gates is not a gate.
- **Status quo / no action.** Rejected: the pre-flip state had one `log.warning` between a total
  authorization bypass and nobody noticing, and — until the same day's alerting fix (AD-133) — no alarm
  in the repository reached a human at all.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A bypass of the receiving authorization chain is detectable within one 5-minute period, at critical severity | One more always-on alarm on the bill, on a metric that should read zero forever (the AD-24 shape: any datapoint is a defect, not a business event) |
| The `ENFORCE` flip becomes a bounded, observable step rather than a leap | Four pending flips are now gated on per-caller instrumentation that does not exist yet for the other engines |
| The failure is classified at the point where the distinction is knowable — a Cedar DENY no longer reads like a network timeout | Every future degrading caller in front of a control inherits a classification obligation, which is real work that looks like boilerplate |
| Reuses AD-46's existing `procurement/resilience` + `reason` pattern rather than inventing a mechanism | The metric proves a bypass happened, never that one did not: a fallback that fails before emitting is still silent |

The alarm is a detector, not a control. It does not prevent the bypass, and a sufficiently broken
environment could take the fallback on the very first PO after a flip. What it buys is that the state is
no longer indistinguishable from healthy — which, for a control whose whole purpose is to be relied upon,
is the difference between a risk and an illusion.

## Results

Realized in impl PR #333: `orchestrator/po_delivery.py` (`_classify_gateway_failure`, the
`po_delivery.gateway_fallback` emission), `infra/bucket_a_alarms.tf`
(`aws_cloudwatch_metric_alarm.po_delivery_gateway_auth_bypass` → `alerts_critical`), and ten tests pinning
that a 403 still returns `RECEIVED` — which is exactly why it must emit a metric rather than surface as an
error.

This decision is why AD-40's production rollout now creates the receiving engine in `LOG_ONLY`: the flip
is a separate deliberate deploy, and this alarm is its prerequisite. It applies unchanged to the four
engines added by impl PRs #326/#328/#329 (`pr_ingest`, `sfn_orchestrator`, `tenant_mdm`, `master_data`),
none of which has a live caller yet — so for those it composes with AD-148's precondition rather than
replacing it. AD-46 carries the pointer from the resilience side, since the obligation lands on degrading
callers generally, not on Cedar specifically.

Open: the four remaining engines have no caller-side classification. When the live Skill-runtime path is
wired through a Gateway (AD-148's addendum), that caller acquires this obligation before its engine can
leave `LOG_ONLY`.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
