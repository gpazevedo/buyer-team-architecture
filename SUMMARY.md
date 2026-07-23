# Buyer Team Architecture — Conceptual Summary of the ADRs

A distillation of the knowledge captured across 131 architecture decision records for the
Buyer Team agentic procurement platform. This is conceptual: it explains the ideas the
decisions encode and how they hang together, not the implementation details.

---

## 1. The root idea: deterministic orchestration wraps non-deterministic intelligence

Everything descends from one decision (AD-001): a governed, audited business process
cannot be run by an LLM that decides its own next steps. The system is therefore a
**hybrid two-level topology** — a deterministic Step Functions DAG owns the workflow
shape, records every transition, and *invokes* LLM agents at fixed nodes; intelligence
never invokes the orchestrator. The workflow is a fixed seven-node graph with exactly one
conditional branch (the Kraljic quadrant router) and exactly one governed loop (a single
cycle-back from human review, after which the negotiation escalates rather than looping
again). The cost of this rigidity is accepted deliberately: a new negotiation style
requires a DAG change, but a misbehaving agent can never alter the workflow shape, which
is what makes 100% governance-compliance and audit-completeness targets achievable.

The same philosophy governs state: the orchestrator's DynamoDB record is the long-lived
truth; agent sessions are ephemeral execution windows. Agents never talk to each other —
they communicate only through the orchestrator's shared state — and they hold no
cross-session state of their own. Where an LLM turned out to add no value over
deterministic code, it was removed: bid evaluation became inline scoring, low-value spot
bids skip the agent entirely, rejection letters are template renders, and structured
responses are assembled in code from the tool-call transcript rather than by a second
model call. The recurring test is *"does this step need judgment, or just correctness?"*
— and only judgment earns a model invocation.

## 2. Agents as bounded specialists; control lives outside the prompt

Each of the six agents owns exactly one cognitive domain and runs as its own isolated
runtime. Three principles bound what an agent can do:

- **Tools as boundaries.** Every external action is a tool call; the system prompt
  carries reasoning instructions only. Data in and side effects out flow exclusively
  through typed, idempotent tools.
- **Steering over prompting.** Behavioral guardrails are runtime hooks that intercept
  tool calls *outside* the reasoning loop — the model cannot forget them, be talked out
  of them, or be injected past them. Hook failures are fail-closed: a crashed guard
  suppresses the tool rather than letting it run unchecked.
- **Governance in code, not prompts.** Every policy — spend gates, budget ceilings,
  approved-supplier filters, ESG rules, the walk-away price — is enforced by
  deterministic code at a specific node. The agent owns the decision *within* the
  permitted range; the orchestrator owns the range. "Never reveal the reserve price" is
  not an instruction; it is structurally impossible because winner fields are not
  template parameters.

## 3. Configuration as data, with a disciplined fail-fast doctrine

All dynamic configuration lives in one DynamoDB table, read once at agent instantiation
— no polling, no mid-session change, changes take effect on the next construction. A
single factory is the sole assembly point for every model request, which lets it enforce
invariants (like cache-prefix purity) by test rather than convention. Thresholds resolve
in two stages — per-tenant override, then a named profile — so tenant customization
never forks code.

The doctrine on missing config is deliberately asymmetric. Configuration is
**fail-fast**: if the config plane is unreachable, agents refuse to construct rather
than run on stale or hardcoded values. But there are exactly **two bounded, documented
exceptions**: feature flags fall back to their seeded safe defaults (so a config outage
can never *relax* a security control), and model-tier resolution walks a never-raise
ladder (config → env override → seed mirror) so a fresh environment can always boot.
Everything else — thresholds, temperatures, governance blocks — still fails fast, and a
missing governance block is distinguished from an unreachable table so an infrastructure
incident is never misdiagnosed as a config-deploy defect. The same registry that maps
logical agents to runtimes also carries canary machinery: per-tenant variant pins, shadow
invocations that measure agreement without affecting results, and deterministic
population splits — rollout as configuration, not deployment.

## 4. Recovery is designed in: idempotency, checkpoints, and honest escalation

The system assumes crashes mid-workflow. State is checkpointed after every node, and
every node and every state-mutating tool is idempotent via explicit dedup keys, so
re-execution detects completed work instead of repeating it. Concurrent recovery is
guarded by a TTL lock sized from worst-case node execution time.

When automation cannot safely proceed, it does not guess: it escalates to a single
`REQUIRES_ATTENTION` status carrying a machine-readable trigger from a numbered taxonomy
(currently 18), each with a defined escalation path and SLA. The escalation signal
itself follows a priority order — the DynamoDB status write is authoritative; the DLQ
publish and its immutable S3 archive are best-effort and can never block it. Fallbacks
are quadrant-aware rather than uniform: a rule-based classification is auto-accepted
only where a wrong answer is cheap, and escalated to a human where it is not.

## 5. Multi-tenancy and security: layered independence, non-spoofable identity

The security model is stated as an invariant: **no single layer's failure produces a
breach or cross-tenant exposure**. Six conceptual layers (infrastructure, identity,
tool/entity access via Cedar, content filtering via Bedrock Guardrails, behavioral
steering hooks, application checks) stand independently, and tenancy specifically is
defended four ways — partition-key namespacing at the data engine, a gateway interceptor
that *overwrites* any client-supplied tenant id with the JWT-derived one (failing closed
on any error), per-request ABAC session credentials, and predicate rewriting at
integration plugins. The model never decides what a tenant can see; the database engine
does.

Identity is made non-spoofable at its sources: a pre-token Lambda normalizes tenant
identity into a single claim drawn only from trusted bindings (a user attribute the
browser can never write, or the authenticated per-tenant machine client), and federated
tenants get a dedicated IdP whose attribute mapping is fixed at onboarding. Human
approval authority is enforced at exactly one place — the interrupt-resume API — with
tenant-configured claims acting as a ceiling that per-request overrides may only narrow.
Tool access is default-deny Cedar generated from one authoritative table, rolled out
log-only before enforcement. Security-critical tables are hardened defensively: write
denied to agents, audited, alarmed, and auto-reverted if a threshold is ever set below
its governance floor.

## 6. Resilience: degrade where optional, refuse where essential

Seven config-driven resilience patterns (retry+jitter, circuit breaker, idempotency,
bulkhead, timeout, DLQ, one shared invocation wrapper) apply at every outbound call
site, and *all* agent invocations flow through a single wrapper so the patterns cannot
be skipped. On top of them sits a per-dependency **degradation tier**: memory failures
set a flag and relax quality thresholds but never block a negotiation; supplier
integrations circuit-break and continue with whoever remains; throttling queues and
backs off. The mirror image of this availability-first stance is the config plane's
fail-fast rule (§3) — the architecture explicitly decides, dependency by dependency,
whether availability or correctness wins.

Backpressure is inbound and honest: each agent caps its in-flight work and sheds excess
with a retryable 503 rather than queueing, while health probes always pass so a
saturated agent never looks dead. A replica's lifecycle is closed at both ends — the
readiness gate opens only after the agent has proven it can serve, and shutdown drains
in-flight work before exit. Bulkhead sizes are derived from platform rate limits and
SLAs, not intuition — including one recorded decision *rejecting* a proposed cap
reduction by showing the arithmetic it would break.

## 7. Observability and evaluation as a closed, self-watching loop

Observability is split into four layers by audience — platform (SRE), application spans
(engineering), domain business metrics (procurement), and FinOps (unit economics) — and
the whole negotiation surfaces as one distributed trace via header propagation enforced
in the shared wrapper. Where no live call exists to carry a header (event pickup, a
human approval pause), correlation state is persisted as plain row data and restored on
resume, so traces survive discontinuities.

Evaluation is matched to the question: LLM-as-judge for qualitative scores, ground-truth
golden sets for accuracy, cheap code checks for structure. Judges deliberately come from
a *different model family* than the agents they score. Coverage is 100% online except
where volume forces tiered sampling, and scores are wired to automated consequences —
model rollback, auto-send disable, auto-award block — closing the loop from measurement
to action. The observability system also watches itself: metric emission failures emit a
non-recursive "I failed" datapoint, and a heartbeat dead-man's-switch alarms on *absence*
of data. Equally characteristic is scope honesty: what is built, stubbed, or deferred is
recorded explicitly rather than left as implied-done design.

## 8. Cost as an architectural dimension

Cost is engineered, not observed after the fact. Each agent runs on the cheapest model
tier that passes its quality bar, with the tier→model mapping in config so corrections
ship without code. Prompt caching rests on a tested invariant — only invocation-invariant
content precedes the cache checkpoint. Expensive work is systematically collapsed:
classification results are semantically cached with threshold-aware keys that
self-invalidate, supplier communications are O(1) in supplier count (one LLM call
produces a canonical body; per-supplier copies are deterministic renders), keep-alive
pings exit before touching the model, and oversized tool outputs are truncated
head+tail. Ground truth for spend is the AWS bill itself: token-based estimates supply
only the proportional breakdown, scaled to the billed total, attributed per tenant.

## 9. Tenants as first-class lifecycle objects with fair, atomic admission

Capacity is a shared pool with per-tenant guarantees: each tenant holds a reserved floor,
a hard ceiling, and a burst weight, under a global invariant that reservations never
exceed platform capacity. Admission is one atomic transaction — status check, counter
increments, and state transition succeed or fail together — while the spend gate is a
deliberately non-atomic pre-check with a *bounded and quantified* overshoot, because
budget timeliness matters more than budget exactness. Concurrency and spend enforcement
have independent kill-switches; a reconciler corrects counter drift on a schedule (and
deliberately skips the budget counter, which has no observable ground truth and fails
safe). Tenants move through an explicit six-state lifecycle: onboarding is a
compensating, dry-run-validated workflow; suspension rejects new work but drains
in-flight; purge leaves a permanent tombstone reserving the identity.

## 10. Integration, platform, and delivery discipline

External systems connect through one **Skill** (all logic) exposed by thin **Plugins**
(pure transport declarations) over four transports with a stated preference order; the
same skill serves any ERP via per-tenant configuration. PO export is decoupled from
awarding through a durable outbox — the procurement decision never waits on a partner
system. Tool surfaces use progressive disclosure (catalog → manual → invocation) so
agents load capability detail only when needed.

Delivery follows build-once-promote: images built on merge move unchanged through
environments, and rollback *is* a roll-forward to a known-good SHA. Infrastructure is
Terraform-first, with platform API gaps closed by explicitly documented provisioner
exceptions rather than silent workarounds; stateful resources carry `prevent_destroy`
so an automated apply physically cannot delete data; mutable-but-dangerous platform
operations get guarded paths that make the wrong call impossible rather than relying on
care. Testing is a five-layer pyramid where each layer gates a different promotion, and
model-behavior evals run on every PR against version-pinned golden data.

## 11. The meta-lessons

Reading across all fourteen themes, a few convictions recur:

1. **Trust structure, not the model.** Determinism, schemas, hooks, and database
   constraints do the enforcing; the LLM contributes judgment inside code-owned bounds.
2. **One owner per invariant.** The factory owns the request shape, the wrapper owns
   resilience, one API owns approvals, one table owns config — invariants live where
   they can be tested once.
3. **Decide failure semantics per dependency.** Fail fast where wrong answers are
   dangerous; degrade gracefully where partial answers are useful; make every exception
   bounded, named, and written down.
4. **Fail closed at boundaries, best-effort behind them.** Security paths refuse on
   error; telemetry and exports never block the business outcome.
5. **Record reality, including corrections.** ADRs are amended when live behavior
   contradicts the design, deferred work is labeled deferred, and dormant mechanisms
   ship dark until deliberately cut over — the record's value is that it stays true.

---
*Summary of the [Buyer Team architecture](https://buyer-team.com) decision record ·
by Gustavo Peixoto de Azevedo*
