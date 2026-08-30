# Buyer Team ADR Digest — All 153 Decisions, One Entry Each

A concise per-decision digest of all 153 Architecture Decision Records, grouped by category.
Each entry condenses five dimensions from the ADR itself: the problem that forced the
decision, the alternatives considered and why they were rejected, the trade-offs accepted,
the decision, and its realized results. Status reflects the ADR files as of 2026-08-27
(reconciled through impl PR #384).

## Contents

1. [Orchestration, State & Recovery](#1-orchestration-state--recovery)
2. [Agent Architecture & Behavioral Control](#2-agent-architecture--behavioral-control)
3. [Dynamic Configuration & Agent Factory](#3-dynamic-configuration--agent-factory)
4. [Multi-Tenancy & Isolation](#4-multi-tenancy--isolation)
5. [Security, Governance & Trust Boundaries](#5-security-governance--trust-boundaries)
6. [Reliability, Resilience & Graceful Degradation](#6-reliability-resilience--graceful-degradation)
7. [Observability & Evaluation](#7-observability--evaluation)
8. [Cost Architecture & Optimization](#8-cost-architecture--optimization)
9. [Infrastructure, Deployment & Platform Stack](#9-infrastructure-deployment--platform-stack)
10. [Capacity, Admission Control & Tenant Lifecycle](#10-capacity-admission-control--tenant-lifecycle)
11. [Integration, Skills, Plugins & Transports](#11-integration-skills-plugins--transports)
12. [Procurement Domain Logic](#12-procurement-domain-logic)
13. [Test Tenant & Platform Data](#13-test-tenant--platform-data)
14. [Test Strategy & Quality Gates](#14-test-strategy--quality-gates)

---
## 1. Orchestration, State & Recovery

### AD-001 — Adopt a Hybrid Two-Level Topology: Orchestration Before Intelligence
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-001

- **Problem:** Procurement requires reproducible execution, enforceable guardrails, and a complete audit trail (OKR O4: 100% governance compliance, 100% audit completeness), but LLM control flow is non-deterministic — a self-planning agent cannot guarantee any of these.
- **Alternatives:** Fully autonomous agent-as-planner (rejected: untraceable, unreproducible workflow changes); static rules-only automation (rejected: cannot handle negotiation variability).
- **Trade-offs:** Gained deterministic, replayable control flow and audit-complete history; given up workflow flexibility (new negotiation styles need DAG changes) and a second control plane (Step Functions + AgentCore) to operate.
- **Decision:** Hybrid two-level topology — a deterministic Step Functions DAG owns the workflow shape and records every transition; LLM agents supply intelligence at fixed nodes, invoked by the orchestrator, never the reverse.
- **Results:** Realized as the seven-node DAG (AD-11), single cycle-back (AD-12), governance in code (AD-18). The root decision the rest of the architecture hangs from.

### AD-002 — Step Functions Owns the Lifecycle; AgentCore Sessions Are Ephemeral
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-001

- **Problem:** AgentCore sessions auto-terminate after 15 minutes of inactivity, but negotiations run days or weeks (Strategic target ≤ 28 days); one mechanism cannot serve both timescales.
- **Alternatives:** Sessions as primary lifecycle holder (rejected: 15-min timeout incompatible with procurement timescales); keep-alive polling/heartbeats (rejected: fights platform semantics, still unreliable).
- **Trade-offs:** Gained platform-fitting design and clean recovery via durable state outside the agent; given up in-session continuity and added per-call context-rehydration plumbing/latency.
- **Decision:** The Step Functions execution owns the long-lived lifecycle; AgentCore sessions are short-lived execution windows, one new session per agent invocation.
- **Results:** Forces AD-3 (durable state lives elsewhere). Corrected 2026-08-04 (PR #257): the per-step DynamoDB checkpoint claim was untrue until then — `save_checkpoint()` had zero callers; now wired into all six node Lambdas.

### AD-003 — Cross-Invocation State in DynamoDB, Not AgentCore Memory
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-001

- **Problem:** AgentCore Memory can persist across sessions, but audit, query, and orchestration need structured, queryable, Step-Functions-readable state that opaque semantic memory cannot provide.
- **Alternatives:** AgentCore Memory as authoritative store (rejected: opaque, scoped to actor/session, not queryable by Step Functions); RDS/Aurora (rejected: server-based dependency; DynamoDB's SFN integration and serverless scaling fit better).
- **Trade-offs:** Gained structured, queryable, audit-complete state; given up the platform's semantic-memory convenience and 21 DynamoDB tables to model explicitly.
- **Decision:** AgentCore Memory only for turn-by-turn context within one invocation; all cross-invocation state (lifecycle, bids, awards, supplier history) lives in DynamoDB.
- **Results:** Cross-negotiation knowledge deliberately excluded here becomes a separate independently-degradable subsystem (AD-72) rather than an implicit platform feature.

### AD-011 — Seven-Node DAG with a Single Four-Way Branch
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-002

- **Problem:** Negotiation needs deterministic auditability, structural governance, and checkpoint recovery — none satisfiable by a free-running multi-agent system; the four procurement strategies come from the Kraljic 2×2 matrix, so routing is naturally finite.
- **Alternatives:** Autonomous agent swarm (rejected: no deterministic audit path); one graph per strategy (rejected: duplicates shared head/tail, loses converging evaluation); deeper nested branching (rejected: extra maintenance, no routing gain).
- **Trade-offs:** Gained a deterministic, auditable path and tractable checkpoint recovery; given up a fixed strategy set (a fifth style requires a graph change) and constrained Node 4 variant divergence.
- **Decision:** Fixed seven-node DAG; one four-way branch at Node 3 keyed solely on Kraljic quadrant, exactly one Node 4 variant runs, all converge at Node 5 and terminate at Node 7. Routing is deterministic local Python, no LLM (sub-100ms).
- **Results:** Every negotiation has a known replayable path, making flow-adherence a code-based eval and routing a ground-truth eval; enables checkpoint recovery (AD-14) and governance-in-code (AD-18).

### AD-012 — Single Governed Cycle-Back (Node 6 → Node 4x, Max 1)
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-002

- **Problem:** A human approver may want a re-run, distinct from rejection; modeling it as a reject flag conflates "cancel" with "try again", and unbounded re-negotiation risks cost blowout, stalling, and stale-bid contamination.
- **Alternatives:** Reject-with-retry flag (rejected: two intents in one signal); unbounded retries (rejected: no termination); no cycle-back (rejected: forces full new negotiation); cycle-back without bid cleanup (rejected: stale bids corrupt evaluation).
- **Trade-offs:** Gained bounded worst-case cost/duration and a guaranteed-clean Node 5 bid set; given up a possible suboptimal award after one retry and a new SUPERSEDED-cleanup failure surface.
- **Decision:** Three first-class Node 6 outcomes — APPROVED, REJECTED (terminal), CYCLE_BACK (max 1). Cycle-back re-routes through the Strategy Router; all bids marked SUPERSEDED (retried 3×, halt to REQUIRES_ATTENTION on failure); a second CYCLE_BACK raises `cycle_back_exhausted`.
- **Results:** All three outcomes live-validated E2E on dev; worst-case cost/duration predictable. Below-threshold NON_CRITICAL spot bids never reach the gate, so cycle-back is unreachable for them.

### AD-014 — Idempotent Nodes with Explicit Dedup Keys + Checkpoint After Every Node
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-002

- **Problem:** Recovery must resume from known checkpoints rather than restart; a node may crash after side effects (invitations, bids, POs), and naive resume would duplicate them.
- **Alternatives:** Checkpoint only at milestones (rejected: coarser recovery, duplicate side effects); no idempotency, restart-from-scratch (rejected: violates recovery requirement).
- **Trade-offs:** Gained one-node recovery granularity (30s target, no duplicates); given up a DynamoDB write per node and a per-node dedup-key design obligation.
- **Decision:** Checkpoint after every node completion (REQ-G002); every node idempotent via explicit dedup keys — `(tenant_id, category_id, hash(items), deadline)` at Node 1, semantic cache at Node 2, `(negotiation_id, supplier_id, action, round_number)` at Node 4x, existing `evaluation_score` at Node 5, existing `CommunicationLog` at Node 7.
- **Results:** Corrected 2026-08-04 (PR #257): the dedup-key half was real throughout, but `save_checkpoint()` was dead code — no checkpoint was ever written, recovery degraded to replay-from-the-top and DLQ redrive escalated everything with `no_checkpoint`. PR #257 wires `checkpoint_node()` into all six node Lambdas (best-effort, skips early errors); the 30s resume target remains unverified live.

### AD-015 — Concurrent Recovery Lock Scoped to (tenant_id, negotiation_id), 600s TTL
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-002

- **Problem:** After a runtime restart, multiple instances may recover the same negotiation concurrently, racing on state writes and side effects.
- **Alternatives:** No lock (rejected: concurrent corruption); short 5-min TTL (rejected in v1.0.4: stolen from slow-but-healthy holder); no TTL/manual release (rejected: crashed holder blocks forever).
- **Trade-offs:** Gained exactly-one-recoverer and crash self-healing; given up to ~10 minutes failover latency if the holder crashes (TTL tension between failover latency and coverage).
- **Decision:** DynamoDB conditional-write lock keyed `(tenant_id, negotiation_id)` with 600s TTL, sized from worst-case node execution (A2A timeout × retries + backoff + checkpoint budget ≈ 500s, rounded up).
- **Results:** Lock-acquisition failure or lock held past 600s without checkpoint progress raises trigger #7 (`recovery_lock_timeout`). Scope note 2026-08-04: the lock guarded a replay that couldn't occur until PR #257 wired checkpoints; sizing still unverified against live recovery.

### AD-016 — REQUIRES_ATTENTION with Twenty Typed Triggers
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-002

- **Problem:** Many distinct conditions warrant pulling a negotiation out of automation (zero bids, risk breach, timeouts, delivery failures, etc.); ops needs a structured, filterable taxonomy, not one opaque "failed" state.
- **Alternatives:** One status per failure type (rejected: explodes the state machine); single untyped error flag (rejected: no triage, no per-condition SLA).
- **Trade-offs:** Gained sub-code dashboard filtering and per-condition SLAs; given up a living trigger-table maintenance surface (grew 9→20, two growth steps collided on number 19) and recurring cross-PRD count-sync work.
- **Decision:** One REQUIRES_ATTENTION status reachable from any status, carrying machine-readable `entry_trigger` + human-readable `entry_reason`; numbered taxonomy (currently 20) with condition, escalation path, and SLA each; ops resolves to ACTIVE or CANCELLED.
- **Results:** Taxonomy reconciled to 20 on 2026-08-05 (`kraljic_low_confidence` #19, `compensation_incomplete` #20); corrected 2026-08-04 (PR #256) that the 96h timeout and 7-day auto-cancel were spec, not behavior; wiring closure 2026-08-14 (PR #293) — all 20 triggers now have a verified fate: real DynamoDB write or documented reason not to (e.g. #4 is a rolling-window alarm, #7 is a deliberate no-op).

### AD-017 — DynamoDB Status Write Is Authoritative; DLQ Best-Effort + S3 Archive
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-002

- **Problem:** Escalation must be reliable — coupling it to SQS means an outage silently loses escalations; and the DLQ's 14-day retention is too short for the audit horizon of a failed financial negotiation.
- **Alternatives:** DLQ as primary signal (rejected: couples escalation to SQS availability); synchronous coupled status+DLQ write (rejected: DLQ failure blocks the status write); DLQ retention only (rejected: 14 days insufficient).
- **Trade-offs:** Gained escalation that never silently fails (status write survives SQS/S3 outage); given up eventual consistency between status and DLQ message, plus storage cost of the 7-year S3 archive.
- **Decision:** DynamoDB status write is the single authoritative escalation signal; SQS DLQ publish is fire-and-forget ops tooling, decoupled so it can never block the write; each publish tee'd to an immutable Object-Lock S3 archive (7-year retention), also best-effort.
- **Results:** Outages degrade only ops tooling/audit convenience, never escalation; `dlq_publish_failed` metric emitted independently; 7-year audit trail for failed negotiations.

### AD-026 — Tool-Level Idempotency via Dedup Keys / Session Cache
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-003

- **Problem:** Ephemeral sessions plus crash recovery and cycle-back re-invocation would re-send invitations, duplicate proposals, and re-score bids — duplicate supplier communications and ranking drift that changes the award.
- **Alternatives:** Node-level dedup only (rejected: can't protect against duplicate tool calls within a crashed node); in-memory session cache only (rejected: lost when the session terminates).
- **Trade-offs:** Gained safe recovery/cycle-back (no duplicate emails, proposals, POs) and deterministic re-scoring; given up a designed dedup key + lookup per mutating tool and a provisioned 72h-TTL cache table.
- **Decision:** Every state-mutating tool dedups and returns `already_sent`/`already_calculated`; comms dedup against `CommunicationLog`; `calculate_tco`/`assess_supplier_risk`/`score_bid` use a DynamoDB session cache (`score_bid` keyed with input-hash, REQ-A513).
- **Results:** Makes the ephemeral-session lifecycle (AD-2) safe to recover; residual: `generate_award_recommendation` is partial — re-invocation can rank slightly differently, absorbed by the human gate or accepted for auto-award. The "resume partial work after DLQ" row was unreachable until checkpoints shipped (PR #257); not yet exercised against live redrive.

### AD-134 — A Scheduled Sweep, Not the State Machine, Owns the Approval Timeout
**Status:** Accepted · **Theme:** 01 Orchestration, State & Recovery · **PRD:** PRD-002

- **Problem:** The 96h approval ceiling (REQ-G203) and 7-day auto-cancel (REQ-G203a) were unimplemented, and the state machine actively defeated the ceiling — `States.Timeout` in the gate's retry list re-invoked Node 6, which reset the clock; Step Functions has no hook to write the authoritative status row, and G203a's window starts after the execution has ended.
- **Alternatives:** Teach Node 6 to detect timeout re-entry (rejected: preserves the clock-reset bug, still no status write); per-negotiation one-shot EventBridge timer (rejected: resource-count leak); DynamoDB TTL (rejected: deletes, can't transition, 48h best-effort); Wait+Choice polling loop (rejected: can't span an execution boundary).
- **Trade-offs:** Gained a durable 96h bound and G203a reachable after execution end; given up exact-time escalation (96h + one 30-min sweep interval), scan-shaped GSI queries, and lifecycle history split across two systems.
- **Decision:** The state machine owns ending the pause (`States.Timeout` removed, caught into a `ApprovalTimedOut` Succeed state); a 30-minute `requires_attention_evaluator` sweep owns the status transitions via conditional writes (G203 → REQUIRES_ATTENTION trigger #3; G203a → CANCELLED after 7 days), race-safe against concurrent human approval.
- **Results:** Shipped PR #256 (2026-08-04). The sweep then failed silently for 12 days on an IAM GSI-query AccessDenied — fixed with the grant plus an `approval_sweep_errors` alarm (PR #319, 2026-08-16); PR #325 added a general REQUIRES_ATTENTION staleness alert (48h, detection only) with a single GSI query per tick. The 96h/7-day paths remain unverified live (no negotiation has crossed the windows); **2026-08-22 (PR #353) — the sweep was dead for ~16 hours** (ADOT-layer import failure, AD-114), so nothing evaluated PENDING_APPROVAL or REQUIRES_ATTENTION and neither the 96h G203 ceiling nor the 7-day G203a auto-cancel advanced; a sweep that stops running stops bounding anything, but unlike the retried SFN task it replaced it has an Errors alarm (AD-144). Open follow-up: drain any backlog of unevaluated REQUIRES_ATTENTION rows. **2026-08-29 (impl PR #392):** the PR #325 staleness alarm was correct and flapping ALARM/OK every ~15min for an unrelated reason — its 300s period saw no datapoint most windows against the sweep's 30-min publish cadence, and `treat_missing_data="notBreaching"` read each gap as "cleared." Fixed via `treat_missing_data="ignore"`, which holds last-known state across gaps instead; no change to the sweep or its cadence


## 2. Agent Architecture & Behavioral Control

### AD-004 — A2A Protocol; Each Agent Is Its Own AgentCore Runtime
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-001

- **Problem:** Six specialized agents need independent versioning, testing, and replacement without redeploying the whole system; a deployment model (monolith vs in-process vs separate runtimes) must be chosen.
- **Alternatives:** Monolithic multi-skill agent (rejected: coupled release cycles, no blast-radius isolation); in-process tool calls between agents (rejected: no independent versioning, one regression takes down the process).
- **Trade-offs:** Gained loose coupling, independent blast radius, and per-agent tiering/Cedar/eval expressibility; given up network-boundary latency, more runtimes to operate, and the need for a shared resilience wrapper (AD-13).
- **Decision:** Each agent is its own AgentCore Runtime, invoked over A2A (JSON-RPC 2.0, port 9000) via `InvokeAgentRuntime`; every invocation crosses a network boundary.
- **Results:** Mandates the shared invoke wrapper (AD-13); enables single-responsibility decomposition (AD-21) and shared-state-only communication (AD-27); protocol immutability validated at plan time (AD-53).

### AD-021 — Single Responsibility per Agent
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-003

- **Problem:** The workflow spans six cognitive jobs (classify, four strategy variants, communications — seven before bid evaluation went deterministic); these could be one large agent or decomposed per domain.
- **Alternatives:** Single large multi-skill agent (rejected: couples releases, removes per-agent tiering and testability); two-agent split (rejected: strategy still spans four incompatible quadrants, no narrow Cedar surface).
- **Trade-offs:** Gained independent testability/versioning/replacement, per-agent model tiering (~3× to 13–23× savings), narrow Cedar surface, small prompts for cache economics; given up six runtimes to operate, coordination moved to orchestration, cross-agent invariants enforced outside the agent.
- **Decision:** Six agents, one cognitive domain each, independently deployable: Kraljic Classifier, Spot Bidding, Leverage Auction, Bottleneck Negotiation, Strategic Partnership, Award & Communications. Design rule: the Spot agent doesn't evaluate its own bids; no strategy agent sends email.
- **Results:** Makes cost tiering (AD-57) and per-agent Cedar least-privilege (AD-39) expressible. AD-117 is the counterweight: a domain that turns out deterministic loses its agent rather than keeping one for symmetry.

### AD-022 — Tools as Boundaries
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-003

- **Problem:** External actions could be prompt text or typed tool calls; the choice has security, caching, and auditability consequences — prompt-embedded data invalidates the cache prefix and cannot be intercepted by Cedar or steering hooks.
- **Alternatives:** Prompt-embedded actions (rejected: uninterceptable, cache-invalidating, no audit record); hybrid read-tools/prompt-writes (rejected: writes are exactly what needs interception).
- **Trade-offs:** Gained a security interception point (Cedar + steering hooks at the tool boundary), cache-prefix stability (~90% input-token savings), full auditability; given up a designed tool contract/schema/dedup key per capability.
- **Decision:** Every external action is a tool call, never prompt text; the system prompt carries reasoning instructions only; state-mutating tools carry dedup keys and idempotency contracts.
- **Results:** Structural precondition for steering hooks (AD-23/24), cache purity (AD-28), idempotency (AD-26), and Cedar per-agent tables (AD-39). Updated 2026-07-12 (PR #201): bottleneck/strategic agents' tool surface narrowed to side-effecting calls only — TCO/risk lookups precomputed in the orchestrator, an explicitly accepted trade-off of argument-list completeness over calculation provenance.

### AD-023 — Steering over Prompting
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-003

- **Problem:** Guardrails must hold every time (never leak competitor pricing, always TCO before award); prompt instructions measure 82.5% compliance vs 100% for runtime hooks, and prompt text is vulnerable to adversarial injection.
- **Alternatives:** Prompt-only guardrails (rejected: ~1-in-6 violation rate + injection vulnerability); post-processing output filters (rejected: too late — side effects already fired).
- **Trade-offs:** Gained deterministic 100% enforcement and injection resistance; given up code to write/maintain, pre-call latency, and a new failure mode (hooks wrongly rejecting work, which needs its own rejection-handling subsystem).
- **Decision:** Guardrails are Strands Steering Hooks intercepting tool calls at runtime, outside the reasoning loop — the LLM cannot forget, be talked out of, or be injected past them.
- **Results:** The 82.5→100% delta is the core result; depends on AD-22 and drives AD-24's failure semantics. Corrected 2026-08-03 (PR #253): the claimed rejection-count alarm never existed. Reconciled to Strands 1.43: all guards run PRE-CALL in GUIDE mode (the shipped API has no after-tool stage).

### AD-024 — Steering Hook Failure Semantics (6 PRE-CALL GUIDE Guards + 1 Declarative, No Retry-Wrap, Fail-Closed in a Base Class We Own)
**Status:** Accepted — the fail-closed guarantee was unrealized until 2026-08-03; see the correction below · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-003

- **Problem:** Two failure questions: should rejections auto-retry, and should a crashed hook still let the tool run? The second is safety-critical — a `WinnerDisclosureGuard` crash on auto-send would leak winner identity/price — and a third part was missed: who is responsible when the framework's steering handler is fail-open by default.
- **Alternatives:** Retry-wrap rejections (rejected: hides the violation signal); fail-open on crash (rejected on merits but was the shipped behavior until PR #253); patch/fork Strands' handler (rejected: vendored, upgrade path); re-raise to agent-level retry→DLQ (rejected: framework swallows the exception); POST-CALL placement (rejected: too late for auto-send).
- **Trade-offs:** Gained fail-closed security actually enforced in code and one base class every guard inherits; given up availability when a guard breaks (hard stop for its tool) and a `check()`-vs-`steer_before_tool` convention the framework doesn't enforce (now statically tested).
- **Decision:** Seven guards total — six PRE-CALL GUIDE-mode steering hooks plus declarative `BudgetCeilingGuard` (response fields + prompt, since the shipped API can't mutate results). No retry-wrap. Every guard subclasses `buyer_agent_core.steering.GuardHandler`, whose `steer_before_tool` catches escaping exceptions from `check()`, emits `governance.hook_error`, and cancels the call with no-retry guidance.
- **Results:** Fail-closed became true on 2026-08-03 (PR #253) after being asserted-but-false since v1.0.5; pinned by tests including one that proves upstream Strands is fail-open without the base. 2026-08-06 hardening: guard-check timeout (10s), static boundary tests, real-guard failure tests, spec-registration tests; `governance_hook_error` alarm applied. **Closed 2026-08-19 (impl PR #333):** the alarm now reaches a human, and this ADR's 2026-08-06 diagnosis — that the vanished SNS subscription "looks like a state/teardown question" — was wrong. The subscription is `count`-gated on `var.alert_email`, whose documented home was a gitignored `terraform.tfvars`, so every CI apply planned it away; the coincident NAT teardown was correlation, not cause (AD-133 carries the full blast radius). `WinnerDisclosureGuard` is now safe by construction (template renders, winner fields not parameters).

### AD-027 — Agents Communicate Only Through Step Functions Shared State
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-003

- **Problem:** Six agents must pass data along the pipeline; they could call each other directly (mesh), via a bus, or through orchestrator-mediated shared state — the choice interacts with crash recovery.
- **Alternatives:** Direct inter-agent mesh (rejected: service discovery, schema coupling, reconstruct-not-replay recovery); shared message bus (rejected: duplicate ordering/delivery guarantees the orchestrator already provides).
- **Trade-offs:** Gained loose coupling, single source of workflow truth, clean checkpoint/recovery; given up all coordination latency through the orchestrator and a shared-state schema that ripples across all six agents.
- **Decision:** No direct inter-agent communication; each agent reads input from the Step Functions shared-state dictionary and returns output to be merged; concurrency exists only within an agent; agents hold no cross-session state.
- **Results:** The agent-side reflection of AD-1: agents stay stateless and ephemeral, recovery replays state rather than reconstructing conversations, and per-node checkpoints (AD-14) capture full pipeline results.

### AD-117 — Bid-Evaluation AgentCore Runtime Removed; Inline Deterministic Scoring Replaces the LLM Agent
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-002, PRD-003

- **Problem:** The Bid Evaluation agent cost 25–60s of LLM time plus microVM startup per invocation for an inherently deterministic task — checking schema fields, governance compliance in tool-call records, and a weighted composite; no subjective judgment involved.
- **Alternatives:** Keep agent with caching (rejected: near-zero hit rate, structurally synchronous); async evaluator Lambda (rejected: composite gates the approval decision, must complete before SFN advances); smaller model (rejected: no model size helps a deterministic check).
- **Trade-offs:** Gained Node 5 dropping to <1s (~30s off the hot path) and one fewer runtime to operate; given up LLM-judged negotiation quality in the composite (now purely structural; LLM-as-Judge eval is the separate quality signal).
- **Decision:** Remove the `bid_evaluation_llm` runtime entirely; replace with `orchestrator/scoring.py` — inline deterministic `score_bid_evaluation()` in the Node 5 Lambda. Same PR: low-value spot/auction items under $5k skip the LLM entirely (~5s vs ~45s); Spot tier downgraded to Nova Lite.
- **Results:** Shipped PR #151 (2026-07-06); 6 LLM agents remain. Retirement completed 2026-08-03 (PR #255): four stale references (resolver fallback, expected-runtime fixtures, tests, docstrings) outlived the runtime — converting an agent to inline has a code-side checklist, not just a terraform destroy.

### AD-125 — `response_builder` Replaces the Deprecated `structured_output()` Second Bedrock Call
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-003

- **Problem:** Every agent run made a second full Bedrock call (`structured_output()`) purely to coerce the finished transcript into a response schema — doubling cost/latency with no reasoning gain and a second chance to hallucinate a field.
- **Alternatives:** Keep `structured_output()` (rejected: pure added cost); make each tool return the full response (rejected: couples tool contracts to response shape); parse the final free-text message (rejected: reintroduces injection/parsing fragility AD-22 eliminated).
- **Trade-offs:** Gained one Bedrock call per invocation (roughly halved cost/latency) and testable Python assembly; given up the second model pass's incidental self-correction — hallucinated tool arguments now flow straight into the response, and two migrations needed new pass-through tools (real model-facing risk).
- **Decision:** `AgentSpec.response_builder` assembles the Pydantic response in code from the tool-call transcript; where no tool naturally carries the response, a thin pass-through tool forces the judgment through a typed argument (`submit_kraljic_classification`; extended `close_auction`).
- **Results:** All 6 agents migrated across PRs #206/#207/#210; `structured_output()` remains only as unused fallback. Update 2026-07-19 (PR #225): the migration was silently broken in every deployed image for ~a week — Dockerfiles' explicit-filename COPY didn't include `response_builder.py`, so agents crashed at startup and the fleet quietly served stub-priced bids (invisible behind AD-46's graceful fallback); fixed by adding the COPY lines and rebuilding.

### AD-135 — `buyer_agent_core` Is the Agent Layer's Only Seam onto Strands, A2A and boto3
**Status:** Accepted · **Theme:** 02 Agent Architecture & Behavioral Control · **PRD:** PRD-003

- **Problem:** AD-101 claimed a tested boundary ("no agent imports outside `buyer_agent_core`") that never existed; within six weeks the duplication returned — six copies of one botocore Config, six direct `strands` imports, five steering files importing the guard vocabulary from two places. A convention that is never checked is no convention.
- **Alternatives:** Documented convention (rejected: that's what failed); deny-list of known packages (rejected: new dependencies pass silently); ruff banned-api lint (rejected: same deny-list flaw, can't express `__all__` membership); packaging boundaries (rejected: container restructure for a guarantee an AST walk gives free).
- **Trade-offs:** Gained one place to tune connection behavior, no silent new platform dependencies, an explicit `__all__` contract; given up indirection (tool bodies don't show their boto3 config) and an allow-list friction toll. Narrow claim: import containment only, not portability off Strands/Bedrock.
- **Decision:** `buyer_agent_core/aws.py` owns the only boto3 construction (shared Config, cached resource/table accessor, lazy region); `__init__` re-exports the full seam; three structural tests enforce an allow-list of imports, no dynamic-import machinery, and `__all__` membership for every imported name.
- **Results:** Shipped PR #258 (2026-08-05); six `tools.py` and five `steering.py` converted seam-only; 1429 tests pass; lazy-region pinned by test; closes AD-101's correction and strengthens AD-22/23 by making "agent modules are declarative surfaces" checked rather than asserted. **2026-08-25 (impl PR #382) — the import boundary held while the seam leaked around it.** `buyer_agent_core` re-exported `Attr` and `ClientError`, so agent code still built its own DynamoDB scan filters and branched on the store's error type: the boundary was intact and the coupling it exists to prevent was not, because a re-export is a hole every import-based check passes. `bids_for_negotiation()` and `ddb_get_item()` in `aws.py` absorb both escapes (the first replacing a byte-identical `Attr(...).eq(...)` scan duplicated in two `tools.py`; the second collapsing a failed read to `None`, behavior unchanged), and both names are dropped from `__all__` so the existing `__all__` assertion now enforces the rule. Award-comms and leverage-auction end up holding no table handle at all. Generalizable: an import-boundary test is necessary and not sufficient — the seam's own `__all__` is the second surface.


## 3. Dynamic Configuration & Agent Factory

### AD-025 — DynamicAgentFactory and Config-as-Data (Fail-Fast, No Fallback)
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-003

- **Problem:** Agent parameters (model, temperature, thresholds) must be tunable per tenant/environment without redeploys; the old AppConfig + `_COLD_START_DEFAULTS` design let agents run on stale or guessed config, contradicting the single-source-of-truth contract.
- **Alternatives:** `_COLD_START_DEFAULTS` fallback (rejected: silent corruption of governance values; removed v1.0.24); AppConfig with sidecar caching (rejected: extra sidecar/polling without solving staleness; retired v1.7.0); last-known-good cache (rejected: stale config with no signal).
- **Trade-offs:** No-redeploy per-tenant tuning and one source of truth, but read-once semantics (changes wait for next instantiation) and the config plane becomes an availability dependency.
- **Decision:** Every agent is built by `DynamicAgentFactory`, which reads all parameters from `{env}-system-config` at construction; unreachable required config fails fast (no defaults, no degraded agent).
- **Results:** As built (2026-06-21), the factory's scope was narrowed to model-ladder resolution + cache-prefix invariant only (delivered via the `agent-base` image); governance/thresholds/flags moved to the orchestrator, where the AD-48 fail-fast contract and two-stage resolution (AD-64) now actually apply. Enabled AD-65/AD-28; exceptions AD-49, AD-95.

### AD-028 — Prompt-Cache Prefix Purity
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-003

- **Problem:** Bedrock prompt caching (≈90% input-token saving, needed for the <$0.10/negotiation target) only works if the prefix before the cache checkpoint is byte-identical across invocations; per-invocation values leaking in (supplier data, thresholds, memory) force full-cost cache writes, and two agents had already drifted.
- **Alternatives:** Per-tenant system prompts with interpolated thresholds (rejected: fragments cache per tenant); governance rules in the system prompt (rejected: varies with tuning + AD-22/23 violation); memory context in system prompt (rejected: cache miss every invocation); convention-only enforcement (rejected: silently rots — proven by v1.0.23 correction).
- **Trade-offs:** The main cost lever actually fires and data/prompt separation is forced, at the cost of a permanent authoring constraint and silent-regression risk that must be caught by test (REQ-C004).
- **Decision:** The cached prefix is exactly the invocation-invariant `system` prompt + `tools` schema, with an explicit checkpoint at the tools/message boundary; everything per-invocation sits after it, enforced by a prefix-purity test.
- **Results:** Realized in `DynamicAgentFactory` (REQ-C004); AD-22/AD-23 become jointly load-bearing for cost; AD-65 makes the factory the single enforcement point.

### AD-048 — Fail-Fast on System-Config Unavailability
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-006

- **Problem:** All dynamic config lives in `{env}-system-config`, but an agent running on guessed governance thresholds could silently approve spend over limits or disable guardrails — AD-46's availability-first posture cannot extend to the config plane.
- **Alternatives:** Last-known-good sidecar cache (rejected: stale-config risk — why AppConfig was retired); conservative hardcoded fallback (rejected: "conservative" is undefined across interacting thresholds); degrade with a `config_degraded` flag (rejected: wrong governance decisions aren't recoverable after the fact).
- **Trade-offs:** Agents can never run on stale/invented governance values and failures are loud (DLQ → REQUIRES_ATTENTION), at the cost of stopping rather than degrading during a config-plane outage.
- **Decision:** Unreachable `{env}-system-config` after SDK retries ⇒ fail fast with a clear error; no stale cache, no hardcoded fallback, no silent degradation — a deliberate inversion of AD-46 scoped to the config plane.
- **Results:** Realized in the orchestrator's governance reads (`load_governance_config`, `Unreachable` vs `Missing` split) — not agent construction, which is never fail-fast by contract; two bounded exceptions: AD-49 (feature flags) and AD-95 (model-tier ladder).

### AD-049 — Feature-Flag Safe-Defaults Exception to Fail-Fast (Security-Critical Flags Primary)
**Status:** Accepted (scope broadened at PRD-010 v1.4.9 — all feature flags, not only the security-critical one) · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-006

- **Problem:** Under AD-48 a config outage fails everything, but for feature flags the safe posture is known: `kraljic_classification_override_enabled=false` means enforcement on — failing fast there stalls negotiations with no security gain; scattered implicit `.get()` defaults for other flags were fragile.
- **Alternatives:** Uniform fail-fast for flags (rejected: needless stalls when safe default is unambiguous); default security-critical flags to `true` (rejected: outage could relax a safety control); special-case only the two security-critical flags (rejected: fragile per-site implicit defaults); extend to thresholds/model IDs (rejected: "safe" undefined there — recreates `_COLD_START_DEFAULTS`).
- **Trade-offs:** Security posture is monotonic under config failure and the fallback is one auditable dict, at the cost of a named exception to AD-48 and a sync obligation with the seed.
- **Decision:** All `features`-group flags fall back to explicit safe defaults matching `seed_system_config.py` when config is unreachable — the security-critical flag is the load-bearing case; everything else still fails fast.
- **Results:** Recorded in PRD-010 §3.3 / REQ-C003 as the first of two bounded exceptions to AD-48 (second: AD-95); the security-critical flags are exempt from AD-66's removal contract (permanent safety controls).

### AD-063 — Single `{env}-system-config` Table, Read-Once-at-Instantiation
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-010

- **Problem:** Model IDs, governance thresholds, flags, and external rates must change without redeploys; the prior AppConfig design added a sidecar, polling, and rollout machinery that still didn't prevent stale-config failures.
- **Alternatives:** AppConfig with sidecar caching + auto-rollback (rejected: operational surface, weaker integrity controls; retired v1.7.0); env vars baked into the image (rejected: redeploy per change); per-service config tables (rejected: fragmented, harder to audit).
- **Trade-offs:** One simple, IAM-protectable, auditable table with no sidecar, but no built-in gradual rollout for config itself (mitigated by AD-66 lifecycle and AD-44 hardening).
- **Decision:** All dynamic config lives in one `{env}-system-config` DynamoDB table with four groups (`governance`, `model`, `features`, `external-rates`), read once per instantiation — no polling, no mid-session change.
- **Results:** Consumed by `DynamicAgentFactory` at instantiation; fail-fast (AD-48) with flag exceptions (AD-49); feeds two-stage threshold resolution (AD-64) and the feature-flag lifecycle (AD-66); **2026-08-22 (PR #350/#351/#352)** — one table also meant one wasted read per caller: a single node invocation fetched the identical `governance/default` from five call sites (~40% of its DynamoDB calls), now memoized in one per-invocation shared store loading a whole `config_group` per `Query` instead of a `GetItem` per key, and carried across the inline chain (AD-151); 5 `GetItem`s per node → 3 `Query`s live-measured. This table's key schema is what makes the group Query possible — one call returns a `default` base with its `tenant#<id>` overlays (AD-64)

### AD-064 — Two-Stage Threshold Resolution
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-010

- **Problem:** Tenants have different risk tolerances needing per-tenant thresholds, but a flat config forbids tuning while a fully per-tenant config creates an unbounded, hard-to-harden override surface (the override table is itself a privilege-escalation risk).
- **Alternatives:** Single global config (rejected: one-size-fits-all is wrong for some tenants); fully per-tenant config with no shared profiles (rejected: no baseline, bigger attack surface); three-stage resolution with code defaults (rejected: reintroduces hardcoded fallbacks vs AD-48).
- **Trade-offs:** Per-tenant tuning without redeploy and reusable risk presets, at the cost of computed (not directly readable) effective thresholds and a hardened override table (AD-44).
- **Decision:** Resolve thresholds in two stages — per-tenant overrides in `{env}-tenant-evaluation-config`, then a named system-config profile (`conservative`/`default`/`aggressive`), cascading per key.
- **Results:** Resolved thresholds attach to agent metadata for evaluators/steering hooks, never the system prompt (cache purity, AD-65); override table hardened per PRD-010 §5.1 + AD-44.

### AD-065 — DynamicAgentFactory Is the Single Request-Assembly + Cache-Checkpoint Owner
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-010

- **Problem:** The cache-hit invariant (AD-59) depends on byte-identical prefixes, but seven agent authors assembling requests independently cannot reliably maintain it — drift had already happened.
- **Alternatives:** Each agent assembles its own request (rejected: distributes the invariant across seven authors); convention/code-review only (rejected: the rule silently rots without a test); per-tenant prompts with interpolated thresholds (rejected: fragments cache + §1.1 violation).
- **Trade-offs:** The ~90% cache saving becomes a tested property enforced in one place, at the cost of forbidding agents from putting resolved thresholds/tenant values in the system prompt and accepting a re-warm dip after build-determining config changes.
- **Decision:** `DynamicAgentFactory` is the single request-assembly point and sole owner of the cache-prefix invariant, with the checkpoint explicitly placed at the tools/message boundary, enforced by test (REQ-C004).
- **Results:** Structural counterpart to AD-28; together they make AD-59's cache saving a tested system property; a deliberate architectural ripple forcing governance/filters into tools and hooks (consistent with AD-22/AD-23).

### AD-066 — Feature-Flag Lifecycle
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-010

- **Problem:** Without lifecycle discipline, "temporary" flags accumulate as permanent dead conditional branches, and behavioral changes need a controlled promotion path across environments.
- **Alternatives:** No formal lifecycle (rejected: unbounded flag accumulation); immediate removal when always-on (rejected: no rollback window); LaunchDarkly-style external service (rejected: unnecessary external dependency — the DynamoDB table suffices).
- **Trade-offs:** Controlled rollout and a removal contract, at the cost of lifecycle bookkeeping and a mandatory two-cycle buffer before removal.
- **Decision:** Five-phase lifecycle — Introduction (default `false`) → Testing → Staging → Production → Deprecation (removed after 2 always-on cycles); flags evaluated at instantiation only, per the read-once model.
- **Results:** Flags live in the `features` group (AD-63); the security-critical flag is exempt from removal (AD-49); provides the gradual-rollout mechanism AD-63 doesn't have natively.

### AD-095 — A2A Agent Model-Tier Never-Raise Fallback
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-010

- **Problem:** The factory's fail-fast contract (AD-48) physically couldn't run on the agents' init path — a model id must resolve before an agent can serve A2A traffic, so a transient config blip at cold start would block booting entirely.
- **Alternatives:** Uniform fail-fast on init (rejected: blip becomes a full agent outage for no correctness gain); make all A2A agents share one factory (adopted — PR #42, `buyer_agent_core` + `agent-base` image); broaden the exception to thresholds/temperatures (rejected: "safe" is undefined there); hardcode the model id (rejected: forfeits config-as-data).
- **Trade-offs:** Agents boot through config-plane blips, at the cost of a second named AD-48 exception and a ladder that must stay never-raise (CI drift-guard + placement-guard test).
- **Decision:** Model-tier resolution uses a never-raise three-level ladder — live `tiers[<tier>]` → `BEDROCK_MODEL_ID` env override → seed-mirroring hardcoded default — scoped strictly to model-tier resolution on the init path.
- **Results:** Second bounded exception to AD-48 (first: AD-49); since PR #42 all 6 agents share one `resolve_model_id` ladder in `DynamicAgentFactory`; fallback observability closed (PR #220): `resolve_model` returns `(model_id, resolution_source)` emitting `model_resolver.tier_fallback`, surfaced on `/ping` without gating readiness.

### AD-098 — Agent/MCP/Skill Registry Config Group
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory

- **Problem:** Six hardcoded `*_RUNTIME_NAME` constants scattered across orchestrator nodes meant no single source mapping a logical agent to its deployed AgentCore runtime; adding an agent meant touching node files + seed script + Terraform, and MCP servers/skills had no discoverability.
- **Alternatives:** (None formally listed — the pre-registry hardcoded-constants design was the status quo being replaced.)
- **Trade-offs:** One source of truth for agent identity plus an operational env-var escape hatch, at the cost of a 3-tier resolution to reason about — later sharpened when a stale legacy entry silently resurrected a dead runtime.
- **Decision:** Add a 5th `registry` config group to `{env}-system-config` declaring agents (runtime_name, protocol, capability, model_tier), mcp_servers, and skills; `resolve_agent_runtime_name(logical)` applies env override → registry → legacy default precedence.
- **Results:** All 6 orchestrator nodes + the accuracy harness resolve through `agent_runtime_arn()`; updated 2026-08-03 (PR #255): agent count is 6 (bid_evaluation retired, AD-117) and the stale legacy entry was removed so unknown agents raise instead of resolving dead runtimes; the 3-tier precedence is superseded by AD-131's variant tiers.

### AD-101 — Agent Base Image + Shared Package Delivery via Immutable `agent-base`
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-003/PRD-010

- **Problem:** Seven agents each vendored `config.py`/`observability.py` and independently installed the same 400MB+ dependencies — duplication drift (one missed copy = divergent logic), unpinned dependency versions, and 7× build waste.
- **Alternatives:** Per-agent duplication with a CI copy-guard (rejected: catches drift but keeps 7× maintenance/build waste); monorepo-wide shared package outside the agent tree (rejected: breaks Docker build context); plain `python:3.13-slim` per agent (rejected: reverts to duplication); mutable `latest` tag (rejected: not reproducible).
- **Trade-offs:** One canonical copy of the agent runtime stack, 40-line Dockerfiles shrunk to ~10, and immutable SHA-tagged reproducibility, at the cost of a new base-image CI step and strict `AgentSpec` conformity.
- **Decision:** Ship an immutable `agent-base:<git-sha>` image + shared `buyer_agent_core` package (factory, model_resolver, cache, seed mirror, spec, serve, observability); all agent images `FROM` it and COPY only thin per-agent files.
- **Results:** Implemented in PR #42; correction 2026-08-05 (PR #258): the credited import "placement-guard" never existed — the first enforcing test (`test_agent_layer_boundary.py`) landed then, closing real duplication (6 copies of a boto3 retry policy, 5 divergent steering files); AD-135 records the boundary decision and AST guard.

### AD-131 — Tenant-Scoped Variant Rollout via Registry Variant Routing
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-010

- **Problem:** AgentCore has no fractional traffic routing (AD-56), so the existing observation window is only a monitoring window — it can't answer whether a new model/prompt performs better on real tenant traffic, or expose it to a controlled subset first.
- **Alternatives:** Wait for AgentCore fractional routing (rejected: no committed timeline, and app-layer ARN resolution already suffices); A/B split without tenant pins (rejected: operators need to choose specific tenants first); a separate routing proxy (rejected: duplicates state; the existing resolver is on every invocation path).
- **Trade-offs:** Real tenant-traffic-validated variant rollout despite no platform primitive, at the cost of an operator-driven mechanism (no automatic analysis/roll-forward), fail-fast on misconfiguration, and a "variant" concept alongside AD-56's observation window and AD-153's Synthetics canary — the three disambiguated by the naming decision.
- **Decision:** Extend the registry resolver with three additive layers: explicit tenant pin (`variant`), async shadow self-invocation recording agreement, and deterministic `ab_split` (sha256 hash of logical+tenant for stable buckets); precedence env override > pin > ab_split > base runtime.
- **Results:** Live-verified end-to-end on dev 2026-07-22 (pins, shadow, 30% split over 2000 synthetic tenants, fail-fast); two live-only gaps fixed same-day (unseeded variants block, missing shadow IAM grant PR #244); consequence found 2026-08-03: doubling runtime count crossed the unpaginated `list_agent_runtimes` 10-per-page boundary (fixed via paginator, AD-13 update). Renamed *variant rollout* in the 2026-08-27 naming disambiguation; `canary-by-tenant` and `promote_tenant_variant.py --rollback` are retired and enforced by the `pr-checks` naming gate (impl PR #384), while the live AgentCore runtime *labels* deliberately keep the old spelling — renaming them destroys and recreates six runtimes.

### AD-151 — Per-Invocation Config Cache, Shared by Both Readers, Carried Along the Inline Chain
**Status:** Accepted · **Theme:** Dynamic Configuration & the Agent Factory · **PRD:** PRD-010

- **Problem:** REQ-R405 forbids holding config across invocations, but nothing required re-fetching the *same* item within one — and a node did, from five call sites (`retry`, `agent_invocation`, `circuit_breaker`, `timeouts` via `resilience/config`, plus `graph_common._fetch_config`): ~28 of 69 `GetItem`s per traced negotiation (~40%) were the identical `governance/default` blob, each also an X-Ray subsegment against a trace payload that had just proven to be a hard 500 KB budget (AD-31, AD-29). The config is invariant across the inline chain, yet Nodes 1→2→3→5 re-fetched it seconds apart.
- **Alternatives:** Warm-container TTL cache (rejected: directly contradicts REQ-R405); put the cache inside one reader (rejected: forces `comms_delivery` to import `graph_common` and imposes one transport policy on two readers with deliberately different fail-fast/retry semantics); read once in Node 1 and pass only resolved values (rejected: leaves Node 6 unable to re-read after a human-review pause and no fall-through for uncovered keys); status quo (rejected: 40% of DynamoDB calls and their spans bought nothing).
- **Trade-offs:** 5 `GetItem`s per node → 3 `Query`s and a smaller trace, at the cost of a second freshness boundary to reason about (`clear()` must run before every handler), config *data* now riding the SFN payload (~4.3 KB against a 256 KB cap), and a cache/transport split a future reader may "simplify" back into one reader.
- **Decision:** A new `orchestrator/config_cache.py` holds one store keyed `(config_group, config_key)`, below both readers and importing neither, loading a whole group per `Query` so an absent tenant overlay costs no read; `handle_config_errors` calls `clear()` at the top of every invocation and that call *is* the REQ-R405 boundary; Node 1 dumps the cache into `_config_snapshot` and Nodes 2/3/5 prime from it via `trace_helpers`, while **Node 6 deliberately never primes** (its threshold check may follow an arbitrarily long human-review wait).
- **Results:** Shipped across impl PR #350 (per-invocation memoization + autouse test fixture), #351 (`_config_snapshot` through Nodes 1→2→3→5), #352 (shared store, Query-by-group, flat snapshot); live-measured 3 Queries per node, 633 tests green, and the last −4% of the −54% trace-payload reduction. One IAM lesson: a read-shape change is an IAM change — `comms_delivery`'s role was `GetItem`-only on the config table and needed `dynamodb:Query` added. Snapshot shape is version-fragile by design, so a node reading an older shape simply reads live.


## 4. Multi-Tenancy & Isolation

### AD-006 — Tenant Isolation via Partition Keys + JWT Claim + Context Injection
**Status:** Accepted · **Theme:** Multi-Tenancy & Isolation · **PRD:** PRD-001

- **Problem:** Cross-tenant exposure is the highest-severity failure class, and prompt-based or app-layer-only checks can be ignored, forgotten, or subverted by an injected or confused agent.
- **Alternatives:** Prompt-based isolation (rejected: the LLM is an untrusted enforcement surface); application-layer-only checks (rejected: share the app's failure surface, no structural guarantee).
- **Trade-offs:** Structural isolation that holds even under injection, at the cost of permanent `tenant_id` key-design constraints and the need for additional layers beyond partition keys.
- **Decision:** Enforce isolation at the data engine: `tenant_id#…` namespaced partition keys, a `tenantId` JWT claim on requests, and tenant context injected into the execution environment — the database, not the model, decides what a tenant can see.
- **Results:** Innermost layer of the four-layer defense-in-depth stack (AD-38); supplemented by AD-37 (ABAC), AD-39 (Cedar), AD-70 (predicate rewriting); claim shape normalized by AD-42.

### AD-037 — Per-Request Tenant ABAC Credentials
**Status:** Accepted · **Theme:** Multi-Tenancy & Isolation · **PRD:** PRD-005

- **Problem:** Neither per-Runtime IAM nor per-tenant Skill Runtimes prevent one Runtime acting for tenant A from touching tenant B's data on a shared Plugin — the only prior defense was buggy-or-compromisable Plugin code.
- **Alternatives:** Application-layer Plugin checks only (rejected: bypassable by compromised Plugin code); per-tenant Runtimes with no shared Plugins (rejected: prohibitively expensive at scale).
- **Trade-offs:** IAM-layer protection independent of Plugin correctness and a reduced credential-leak blast radius, at the cost of per-request `AssumeRole` latency and a new `assume_role_failure` failure mode — and it only covers tenant-partitioned targets.
- **Decision:** The Gateway Interceptor assumes a per-Gateway ABAC role with `tenant_id` as an STS session tag, forwarding scoped temporary credentials to the target Plugin (pattern from aws-samples SaaS workshop). **Implementation status: deferred Phase-2** — roles provisioned dormant, interceptor not yet assuming them.
- **Results:** Five dormant per-Gateway roles + `abac_tools` registry + `security.abac.assume_role_failure` alarm wired, but inert (`ABAC_TOOL_ROLES={}` as of PR #43); until wired, AD-41/AD-6/AD-70/AD-39 carry isolation.

### AD-038 — Defense in Depth, Not Replacement
**Status:** Accepted · **Theme:** Multi-Tenancy & Isolation · **PRD:** PRD-005

- **Problem:** When a stronger control (ABAC) arrives, the natural pressure to retire older controls would create a single point of failure — one bypass in the remaining control becomes a breach.
- **Alternatives:** Single canonical isolation mechanism (ABAC only) (rejected: one bypass = direct cross-tenant exposure); remove partition-key checks once ABAC is in place (rejected: ABAC doesn't cover non-partitioned targets).
- **Trade-offs:** The cross-tenant guarantee survives any single layer's failure, at the cost of multiple overlapping checks to maintain and more places for tenancy bugs to hide.
- **Decision:** ABAC supplements — never replaces — partition-key checks, Cedar, and predicate rewriting; invariant: no single layer's failure produces cross-tenant exposure.
- **Results:** Codified as a named principle cited by AD-37 and the integration layer; the four-layer stack is AD-6 → AD-37 → AD-39 → AD-70; future tenancy controls must state their layer.

### AD-041 — Gateway Request Interceptor Rewrites tenant_id, Fails Closed
**Status:** Accepted · **Theme:** Multi-Tenancy & Isolation · **PRD:** PRD-005

- **Problem:** Cedar evaluates the policy principal, not the `tenant_id` argument an agent emits — a prompt-injected agent under tenant B's JWT could pass `tenant_id=A` (the confused-deputy bug) and Cedar wouldn't catch the mismatch.
- **Alternatives:** Cedar principal-condition on the argument (rejected: Cedar can't enforce that an agent-supplied value matches the JWT-bound tenant without a trusted rewrite); Plugin-side argument validation (rejected: shares the Plugin's failure surface).
- **Trade-offs:** The value Cedar and the Plugin see is always the JWT-bound tenant, at the cost of a mandatory Lambda hop on every Gateway-routed tool call and a fail-closed 500 for all tool calls if the interceptor breaks.
- **Decision:** A Lambda interceptor on every Gateway decodes the validated JWT, overwrites `params.arguments.tenant_id` with the normalized `tenantId` claim (AD-42) before Cedar evaluates; missing claim → 403, any failure → 500, target never runs.
- **Results:** Emits `security.interceptor.missing_claim` / `security.interceptor.failure` metrics; structural fix for the confused-deputy gap in the AD-38 stack; body-handling contract per target type is AD-102.

### AD-102 — Gateway Interceptor Built to the HTTP-Target Contract
**Status:** Accepted · **Theme:** Multi-Tenancy & Isolation · **PRD:** PRD-005

- **Problem:** The interceptor's event shape depends on Gateway target type — native MCP targets deliver a parsed `event["mcp"]` body, but our ingest Gateway fronts a Runtime (HTTP target) with a base64-encoded `event["http"]` body; the reference interceptor keyed on the MCP shape would silently no-op on the one Gateway carrying live tenant traffic.
- **Alternatives:** Reuse the PRD-005-impl MCP reference sample as-is (rejected: inert at runtime on an HTTP-target Gateway); convert the ingest Gateway to a native MCP target (rejected: re-architects the live ingest path for no isolation benefit).
- **Trade-offs:** AD-41's rewrite actually runs on the live ingest Gateway, at the cost of target-type-specific code paths and coupling to the GA interceptor event schema.
- **Decision:** Build the interceptor to the GA HTTP-target contract: read `event["http"]`, base64-decode, rewrite `tenant_id` in the JSON-RPC envelope, re-encode, return `transformedGatewayRequest` (or fail-closed response); `pass_request_headers = true`.
- **Results:** Live-validated on dev in PR #43 (Increment 2d); the concrete realization of AD-41 on the ingest Gateway; would carry AD-37's ABAC when wired.

### AD-107 — AgentCore Context Middleware for FastMCP
**Status:** Accepted · **Theme:** Multi-Tenancy & Isolation · **PRD:** PRD-005

- **Problem:** AD-6 requires tenant context injected into the execution environment, but the 4 MCP servers use FastMCP (bare Starlette app) — no AgentCore Runtime header processing, so no per-request identity token or tenant context reaches tools.
- **Alternatives:** Switch to `BedrockAgentCoreApp` (rejected: loses FastMCP's auto-generated JSON-RPC schemas from type hints); manual header extraction per tool (rejected: boilerplate everywhere); status quo (rejected: violates AD-6, blocks ABAC derivation).
- **Trade-offs:** Complete context injection on FastMCP servers via one reusable middleware, at the cost of a new build-time dependency (`bedrock-agentcore` v1.15.1) and wiring carried on local-dev-only servers (safe no-op).
- **Decision:** A reusable Starlette ASGI middleware (`AgentCoreContextMiddleware`) bridges AgentCore Runtime headers into `BedrockAgentCoreContext`, wired first on every FastMCP app; no-op when headers are absent.
- **Results:** Wired into all 4 MCP servers (skill_runtime, dynamodb_master_data, step_functions_orchestrator, tenant_mdm_emulator) on the innermost app; enables JWT claim extraction and the ABAC path (AD-37); 6 unit tests + 163 regression tests pass. Now wrapped inside the broader `build_mcp_app` factory (AD-146).

### AD-119 — Per-Tenant Cognito M2M App Client for PR Event Router
**Status:** Accepted · **Theme:** Multi-Tenancy & Isolation · **PRD:** PRD-007, PRD-005

- **Problem:** All tenants shared one `pr_event_router` M2M App Client, so the pre-token normaliser (AD-42) injected the same `tenantId` into every tenant's JWT — Blue Jets PRs carried `tenantId=test_tenant`, breaking isolation at the Gateway interceptor (AD-41).
- **Alternatives:** One multi-tenant client with a custom claim parameter (rejected: client-credentials grants can't pass a per-request tenant hint); separate User Pool per tenant (rejected: that's the human-login federated IdP path, AD-094); derive tenant from scope (rejected: conflates authorization with identification).
- **Trade-offs:** Correct per-tenant identity on every M2M JWT, at the cost of O(N) clients per tenant (new client + binding row + `COGNITO_CLIENT_MAP` entry per tenant) and a sync obligation on that map.
- **Decision:** One dedicated Cognito M2M App Client per tenant; the router selects per-request via `COGNITO_CLIENT_MAP`; token/secret caches are per-client-id; ingest Gateway admits both router clients; empty map fails fast at startup.
- **Results:** Shipped PR #149 (2026-07-06); completes the identity chain per-tenant client → normaliser claim → interceptor enforcement (with AD-42, AD-41).


## 5. Security, Governance & Trust Boundaries

### AD-018 — Governance Enforced in Code at the Node Level
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-002

- **Problem:** Governance policies (spend thresholds, max rounds, ESG, supplier filters, budget ceilings) expressed only as prompt text can be ignored by the model or defeated by prompt injection.
- **Alternatives:** System-prompt instructions (rejected: bypassable, non-auditable); enforcement in a downstream review service (rejected: after-the-fact detection, can't halt at the decision point).
- **Trade-offs:** Tamper-resistant enforcement with full OTEL audit, but new policies need node-level code changes rather than prompt edits.
- **Decision:** Enforce every governance policy in deterministic code at a specific DAG node — spend/quality gates at Node 6, approved-supplier filter at Node 1, two-tier ESG at Nodes 3+5, comms approval flag at Node 7 — never as prompt text.
- **Results:** Injection-proof governance with every decision in the distributed trace; thresholds tune via `{env}-system-config` without redeploy. Layer in AD-36's stack; complements AD-19 and AD-39.

### AD-019 — Entity Access Control Only at the Interrupt-Resume API
**Status:** Accepted (as-built revised 2026-06-20) · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-002

- **Problem:** User-action authorization (who may approve, resolve cases) is a different plane from agent→tool Cedar auth; without one enforcement point, user-authz could leak into agents or vanish.
- **Alternatives:** Checks distributed across agents (rejected: pushes user JWTs into agents); a single global gate with no per-PR override (rejected: real deployments need overrides).
- **Trade-offs:** One testable gate and no user JWTs in agent runtimes, but the gate is a single point of authorization failure; the additive posture leans on PRD-017 dry-run check 7 to keep real tenants gated.
- **Decision:** Enforce user-action authorization exclusively at the Graph's interrupt-resume API: validate JWT, resolve effective claims (per-PR override → tenant default), AND-aggregate, else HTTP 403. As-built: additive, not fail-closed — with no override or default configured the action is authorized, because JWT-level tenant isolation (AD-6) already holds.
- **Results:** Realized in `orchestrator/node_approval_gate.py`; reconciled to live `pk`/`sk` schema (per-PR override on `{env}-requisitions`, tenant default under `tenant_default_claims.po_approve`). Prior "unconfigured claims fail closed" wording superseded.

### AD-036 — Six-Layer Defense-in-Depth
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** An agentic system handling untrusted supplier content and financial decisions has many attack surfaces; no single control covers all, and any single control can fail.
- **Alternatives:** One comprehensive security layer (rejected: no residual protection after compromise); ad-hoc per-feature controls (rejected: invisible coverage gaps, unclear ownership).
- **Trade-offs:** No single-layer failure becomes a breach, with clear per-layer ownership — at the cost of building and operating six layers, with deliberate redundancy (tenant scoping recurs).
- **Decision:** Organize security as six independent conceptual layers — Infrastructure, Identity, Tool/Entity Access, Content Filtering, Behavioral Guardrails, Application — a stack, not a call sequence; invariant: one layer's failure never produces a breach.
- **Results:** Each layer specified in PRD-005 with OWASP/ATLAS mapping; anchors AD-38 (ABAC), AD-39 (Layer 3), AD-43 (Layer 4), AD-23 (Layer 5), AD-18 (Application). Fallbacks document which layers degrade (4–5) and which remain (1–3).

### AD-039 — Cedar Policies for Agent→Tool Access; the §5.1 Table Is Authoritative
**Status:** Accepted — narrowed 2026-08-06 (never applicable to the 6 LLM agents, see the correction below); footprint superseded 2026-08-19: Cedar now governs five Gateways, not PO Receiving alone · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Which agent may call which tool must be enforced outside agent code, with one unambiguous permission source to prevent doc/policy drift.
- **Alternatives:** Prompt-stated restrictions (rejected: injection-bypassable); IAM-only (rejected: can't express per-agent per-tool semantics); wrapping the 6 agents' tools behind a Gateway just for Cedar (rejected: adds a hop and coarse boundary purely to satisfy a mechanism).
- **Trade-offs:** Cedar stays real, deployed enforcement for PO Receiving; for the 6 agents the §5.1 table is a permission matrix with no policy engine behind it — their actual boundaries are tool registration plus steering hooks.
- **Decision:** As corrected 2026-08-06: the 6 LLM agents' tools are Strands-native in-process `@tool`s (AD-135), so no Gateway hop exists for Cedar to evaluate — the original "Cedar at the AgentCore Gateway" decision was never buildable for them. Cedar governs only PO Receiving (`po_receiving.cedar`, one M2M principal, one coarse permit). The §5.1 table remains the authoritative intent record; enforcement for the 6 agents sits in tool registration and the steering-hook layer (AD-23/AD-24).
- **Results:** As corrected 2026-08-06, Cedar was deployed only on the PO Receiving Gateway; §5.1 no longer described as "generated into Cedar policy files." Cedar is Layer 3 in name (AD-36); in practice for the 6 agents, Layer 3's role is filled by tool registration + steering hooks + Bedrock Guardrails (Layer 4). **Footprint superseded 2026-08-19:** the "only PO Receiving" clause held for thirteen days — impl PRs #326/#328/#329 stood up four more policy engines (`pr_ingest`, `sfn_orchestrator`, `tenant_mdm`, `master_data`), so Cedar now governs five Gateways, all `LOG_ONLY` by default (AD-147, AD-148). What survives from the 2026-08-06 narrowing is the part that was never about footprint: the 6 LLM agents' in-process `@tool` calls still have no Gateway hop for Cedar to evaluate (AD-135), and none of the five engines changes that.

### AD-040 — Cedar Rollout Phases Distinct from Per-Environment Mode
**Status:** Accepted — scope narrowed 2026-08-06 to PO Receiving; widened 2026-08-19 to five Gateways (see AD-39's Results); phase sequencing corrected 2026-08-19 · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** How Cedar enforcement is introduced over time (rollout) and which mode each environment permanently runs are different concerns; conflating them makes production cutover unsafe or CI behave inconsistently.
- **Alternatives:** One unified mode promoted through environments (rejected: CI/staging would track production's rollout phase); immediate ENFORCE in prod (rejected: no observation window).
- **Trade-offs:** Safe observed cutover with stable per-environment modes, at the cost of holding two independent axes (temporal phase vs. permanent mode) in mind.
- **Decision:** One-time temporal production rollout (Observe LOG_ONLY → Validate LOG_ONLY+alerts → Enforce) kept explicitly separate from the permanent per-environment mode (LOG_ONLY in CI/dev/staging, ENFORCE in prod).
- **Results:** Three-phase rollout table in PRD-005 §5.2, scoped to PO Receiving only after AD-39's correction; permanent modes governed by PRD-007 §8 / AD-55. **2026-08-19 (impl PR #333) — phase 1 was unreachable in prod by construction.** `prod-deploy.yml` passed `receiving_policy_mode=ENFORCE` under a comment reserving the flip until "current prod LOG_ONLY telemetry has been reviewed" — but that plan step is the only thing that creates prod's policy engine, so it would have created it already-enforcing and the telemetry the go-ahead required could never exist. Both call sites (plan and roll-forward apply) now pass `LOG_ONLY`, making ENFORCE a separate deliberate one-word deploy; a roll-forward must carry whatever mode the plan carries, or a rollback silently changes enforcement posture along with the image tag (AD-54). The ordering is load-bearing, not ceremony: `po_receiving.cedar` is deny-by-default on a single permit conditioned on `principal.getTag("scope")` resolving, and if that tag does not resolve in prod, ENFORCE denies every call while Node 7 does *not* fail — it catches the 403 and degrades to the in-process boto3 write, so POs keep flowing while each one bypasses the Cedar engine, the REQUEST interceptor and the JWT authorizer together. **A misconfigured ENFORCE is weaker than LOG_ONLY and indistinguishable from healthy from the outside** — general enough that it is recorded on its own as AD-149. Made detectable in the same PR: `po_delivery.py` classifies the gateway exception (auth 401/403, http, config, transport) and emits `procurement/resilience po_delivery.gateway_fallback` dimensioned by reason; the new `po-delivery-gateway-auth-bypass` alarm fires on the auth bucket into the `alerts_critical` tier — the same severity class as `governance_confidentiality_violation`, since both mean a control was bypassed. Transport and config are deliberately unalarmed (transient and noisy, and alarms bill per alarm-month regardless of state — AD-133). Ten new tests pin that a 403 still returns `RECEIVED`, which is precisely why it must emit a metric.

### AD-042 — Cognito Pre-Token-Generation V3 Normalizes tenantId
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Tenant identity arrives in different shapes per issuance path (native users vs. federated vs. M2M), and V2 doesn't fire for `client_credentials`, leaving M2M tenant isolation undefined.
- **Alternatives:** Pre-Token V2 (rejected: doesn't fire for client_credentials); resolve tenant from caller-supplied `ClientMetadata` (rejected: unauthenticated, spoofable).
- **Trade-offs:** One claim shape for every issuance path and a non-spoofable M2M binding — but requires the Cognito Essentials/Plus tier, and `tenantId` lands only on ID tokens for user/federated grants.
- **Decision:** A shared Pre-Token-Generation V3 Lambda normalizes tenant identity to a top-level `tenantId` claim from two trusted bindings only: `custom:tenantId` for humans, and the authenticated per-tenant App Client (via `by-app-client` GSI) for M2M.
- **Results:** Prerequisite for the Gateway Interceptor (AD-41) and ABAC (AD-37); onboarding dry-run check #1 asserts M2M tokens carry the claim. AD-108 later surfaced and additively corrected the `custom:custom:tenantId` double-prefix defect in the pool schema.

### AD-043 — Bedrock Guardrails on All Agents
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Supplier bids/emails are untrusted input carrying injection, PII, and goal-hijacking payloads; without filtering at the model boundary, Layer 3/5 controls are the first line of defense.
- **Alternatives:** Bespoke per-agent validation code (rejected: cost, inconsistent coverage); steering hooks alone (rejected: hooks don't filter content that reaches the reasoning loop).
- **Trade-offs:** Managed model-level filter on every input/output — but it's LLM-based and degradable (reconnaissance/staging coverage only "Partial"), so it can never be the sole defense.
- **Decision:** Apply Bedrock Guardrails (PII detection, topic restriction, PROMPT_ATTACK:HIGH, credential regex) to all agents as Layer 4 content filtering, on every input and output.
- **Results:** After fixes in PR #120/#121/#122 (wiring, policy coverage, version publishing) the guardrail is live-effective; 2026-08-05 update closes the fail-open gap: `GUARDRAIL_REQUIRED=true` makes deployed runtimes refuse to serve unconfigured, plus a `guardrail_unconfigured` MI alarm with a real email subscriber. AD-128 fixed false positives without weakening strength; the AR-carrying guardrail is split from the agent path by AD-140's 2026-08-16 update. **2026-08-29 (impl PR #404):** the topic policy had one DENY topic memorising its own three examples; offline `ApplyGuardrail` sweeps found an 87% (52/60) bare-text ceiling with context-poisoning/goal-hijacking framings entirely uncovered. Widened to three DENY topics (added context-poisoning-untrusted-data, goal-hijacking), published as `agent_path` v2 / `buyer_team` v4 in lockstep; ceiling rose to 97% (58/60), every agent's rendered block rate improved, and a 19-prompt false-positive check found only 1 benign over-match ("supplier override").

### AD-044 — Tenant-Evaluation-Config Table Hardened
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** `{env}-tenant-evaluation-config` lets tenants override quality thresholds — a compromised writer could lower thresholds to near-zero and silently disable quality enforcement.
- **Alternatives:** IAM role scoping alone (rejected: still exposes write surface to compromise); detect-and-alert without auto-revert (rejected: changes persist through negotiations).
- **Trade-offs:** Escalation surface closed at IAM layer and self-healing reverts — but routine threshold changes need a break-glass path and an extra Lambda to maintain.
- **Decision:** Harden with four controls: resource-policy write-deny for all agent roles, CloudTrail data events (365 days), a 60-second write-anomaly alarm, and a Streams Lambda reverting any write below the governance floor.
- **Results:** Pattern later mirrored on `{env}-system-config`; floor-revert owned by PRD-010 §5.1 / REQ-C304; closes the ATLAS "Modify Agent Config" escalation surface at the Application layer.

### AD-094 — Per-Tenant Federated IdP for Non-Spoofable Human Tenant Binding
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Interactive human sign-in had no non-spoofable tenant binding equivalent to the M2M App-Client binding, and role/group ClaimRequirements were unsatisfiable for human users in a shared native pool.
- **Alternatives:** Shared native pool with administratively-set `custom:tenantId` (rejected for production: binding rests on provisioning correctness — identity-forgery surface); one pool per tenant (rejected: multiplies management); M2M-only (rejected: forecloses human approvers).
- **Trade-offs:** Binding rooted in the authentication path, symmetric with M2M — at the cost of per-tenant onboarding provisioning and external IdP dependency; the fixed attribute mapping is a security-critical artifact, not a tunable.
- **Decision:** Each interactive tenant gets a dedicated SAML/OIDC IdP registered in the shared pool with `provider_name == tenant_id`, a fixed mapping populating `custom:tenantId`/`userRole`/`groups`, and a per-tenant user App Client restricted to that IdP.
- **Results:** Realized as REQ-S709–S711, provisioned by onboarding (AD-86); completes AD-42's normalization with the non-spoofable human source; makes AD-19's role/group claims satisfiable. The demo tenant deliberately uses the rejected native-pool shape (AD-108).

### AD-100 — Governance Fail-Fast Split — ConfigUnreachable vs GovernanceKeyMissing
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005/PRD-010

- **Problem:** One `ConfigUnreachable` exception masked two distinct failures — the whole governance row absent (infra incident) vs. a required block dropped from the row (config-deploy defect, the recurring "blocks dropped" drift) — and missing blocks were silently tolerated.
- **Alternatives:** Single exception with message discriminator (rejected: parsing error strings); silent default fallback (rejected: could approve spend with no signal); deploy-time validation only (rejected: can't catch out-of-band DynamoDB mutations).
- **Trade-offs:** Two distinct routable failure signals and guaranteed catch of dropped blocks — but two exception types to maintain and callers must declare their `require=` list.
- **Decision:** Split into `ConfigUnreachable` (row absent / DynamoDB unreachable) and `GovernanceKeyMissing` (row present, required block dropped), with callers declaring required blocks via `require=` on `load_governance_config`; both fail fast per-request → node DLQ → REQUIRES_ATTENTION.
- **Results:** Implemented in `orchestrator/graph_common.py` (PR #42); `node_approval_gate` adopts `require=["approval_thresholds"]`; agent boot path stays never-raise per AD-95.

### AD-108 — Demo SPA Interactive Login via Cognito Hosted UI (Native PKCE Public Client)
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-013

- **Problem:** The demo SPA must exercise the real Cognito→JWT→Gateway→ABAC auth path, but M2M binding doesn't fit a browser user and per-tenant federation (AD-94) is over-built for one demo tenant; a latent pool defect (custom attribute stored as `custom:custom:tenantId`) silently routed Hosted-UI logins to the M2M fallback.
- **Alternatives:** Per-tenant federated IdP for the demo (rejected: over-built, no external IdP); dev no-auth only (rejected: never proves end-to-end auth); SPA-set tenant binding (rejected: browser self-assignment is identity forgery); in-place pool attribute replacement (rejected: Cognito attributes are immutable — would force pool replacement).
- **Trade-offs:** Demo exercises the real auth path via one claim-normalization path — but fallback users all map to one tenant (not AD-94-grade binding), and the mis-named attribute stays as dead schema.
- **Decision:** Hosted UI login via a Cognito-native PKCE public App Client; two-tier non-spoofable binding: `custom:tenantId` per-user seam (via `create_tenant_user.py`) with M2M-style `by-app-client` fallback to the test tenant; attribute restored additively (`schema { name = "tenantId" }`).
- **Results:** Live-validated on dev (token carries `tenantId`; 401/200 without/with token); demo frontend now lives in `buyer-team-demo` repo (PR #183, AD-122). Deliberately adopts AD-94's rejected shape scoped to the demo only.

### AD-127 — Build a Self-Sufficient RFC 8693 Token-Exchange Broker for the Delegation Chain
**Status:** Accepted (built, not enabled) · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Delegated PO delivery should carry who acted (approving human + acting agent), but Cognito supports no token-exchange grant and AgentCore Identity OBO only forwards to a downstream IdP that doesn't exist here — while `approved_by` was already captured and dropped.
- **Alternatives:** Cognito token endpoint (rejected: only 4 grant types, none token-exchange); AgentCore Identity OBO (rejected: forwards only, no exchange happens); overwrite Cedar `sub` with acting identity (rejected: forces policy/fixture rewrite); status quo (rejected: delegated actions un-attributable).
- **Trade-offs:** End-to-end attribution with unchanged authorization subject — but a second signing authority to operate, and no live guarantee until the flags flip and TF is applied (built, not enabled).
- **Decision:** Build `lambdas/token_exchange_broker` — own OIDC discovery/JWKS, verifies inbound Cognito M2M tokens, mints its own RS256 token with additive `actingAgent`/`actingUser` claims, `sub` unchanged; thread `approved_by` downstream; everything behind off-by-default flags (`TOKEN_EXCHANGE_MODE`, `REQUIRE_ACTING_AGENT`).
- **Results:** Shipped in PR #223 as code + validated-but-never-applied Terraform; fixed a concurrent cold-start `ResourceExistsException`; the trust-boundary cutover (Gateway discoveryUrl repoint, Cedar `actingAgent` clause, flag flips) is deliberately deferred.

### AD-128 — Precomputed Context Renders After the Cache Point, and Anonymize-Only Guardrail Interventions Are Not Blocks
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Two agents failed real invokes with Guardrail "blocked content" — two coupled bugs: precomputed blobs in the scanned user turn tripped PROMPT_ATTACK:HIGH (MEDIUM-scored, blocked at LOW confidence), and `serve.py` treated every `guardrail_intervened` (including deliberate ANONYMIZE masking) as a hard block.
- **Alternatives:** Lower PROMPT_ATTACK strength (rejected: regresses AD-35 catch rate 100%→55%); move items/suppliers into the unscanned system role too (rejected: blinds these agents to injection the AD-35 suite targets); treat every intervention as success (rejected: real BLOCKs would pass); reconfigure PII to BLOCK (rejected: breaks routine supplier-email flows).
- **Trade-offs:** Agents unblocked without weakening the guardrail or cache-prefix invariant — but a per-agent `context_builder` seam agents must opt into, and trace capture coupled to Strands' streaming shape.
- **Decision:** Render precomputed blobs into the system role after the `cachePoint` (unscanned, cache prefix byte-identical); classify interventions via `is_masking_only_intervention` — secure-by-default, only provably anonymize-only passes; `items`/`suppliers` stay in the user turn.
- **Results:** Shipped in PR #226; 2026-08-16 update (PR #319): directive prompts in the user turn were a second trip source of the same class, moved to `_build_context` after the cache breakpoint.

### AD-136 — Guardrail-Triggered Reconnaissance Detection (REQ-S601)
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** ATLAS Reconnaissance (adversary probing what Guardrails block before staging an attack) was specified as REQ-S601 but never built; the signal existed at the `GuardrailBlocked` raise sites but was never emitted as a metric.
- **Alternatives:** Per-supplier gating per REQ-S601's literal wording (rejected: no request shape is uniformly single-supplier); a new `procurement/security` namespace (rejected: breaks consistency with sibling alarms); independent trace inspection (rejected: duplicates AD-128's classification).
- **Trade-offs:** A real applied signal closing a months-old gap — but aggregate cross-tenant threshold means patient adversaries spreading or pacing traffic stay under the alarm; still doesn't cover ML Attack Staging.
- **Decision:** Emit `security.guardrail.triggered` from both `GuardrailBlocked` raise sites, using AD-128's classification so masking-only interventions never count; alarm `guardrail_triggered_recon` fires at SUM > 5 per 60-minute window on `evaluation_alerts`.
- **Results:** Shipped and applied to dev (PR #263); PRD-005 Reconnaissance row moved None → Covered.

### AD-137 — Memory-Write Authorization via Candidate-Scope Validation, Not a JWT Claim Check (REQ-S608)
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** REQ-S608's spec'd JWT claim check was never buildable (the write path is a Step Functions Lambda with no workload token), and tracing the real path found `_record_supplier_memory` wrote history rows for whatever `supplier_id` the agent's LLM output named — a real memory-poisoning vector.
- **Alternatives:** Build the spec'd claim check verbatim (rejected: dead code — no token and no attacker-controlled `tenant_id`); do nothing (rejected: the orchestrator is trusted but the agent output it writes is the untrusted surface); reject at read time (rejected: poisoned rows still accumulate).
- **Trade-offs:** Closes the demonstrated poisoning vector and matches the pricing path's trust model — but the spec's JWT form stays unbuilt (a gap only if agents ever gain direct memory-write tools), and incident detail lives in logs, not the metric.
- **Decision:** Authorize memory writes against the orchestrator's own trusted candidate set: any agent-supplied `supplier_id` outside `set(sup_to_bid)` is refused pre-write, emitting `security.memory_write.rejected` (fixed cardinality `{agent_name, reason}`) with a `memory_write_rejected` alarm.
- **Results:** Shipped and applied (PR #264); PRD-005 Memory Manipulation row None → Covered; AD-129 updated with a forward pointer to this authorization layer.

### AD-139 — Enable GuardDuty Extended Threat Detection (REQ-S605)
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** REQ-S605 (GuardDuty CRITICAL findings → SNS) was specified but never provisioned anywhere; GuardDuty findings publish only to EventBridge, so the repo's standard metric+alarm pattern needed a new AWS-service-event → metric bridge.
- **Alternatives:** EventBridge → SNS directly (rejected: breaks the uniform CloudWatch-alarm surface); enable every protection plan (rejected: no EKS/EC2/RDS in this account — billing for nothing); CloudWatch Logs metric filter (rejected: findings don't land in Logs by default).
- **Trade-offs:** First AWS-native, account-level threat signal in the alarm surface — but a new always-on billable detector plus Lambda; only Critical-severity findings alarmed (matches REQ-S605's literal text).
- **Decision:** `aws_guardduty_detector` with `S3_DATA_EVENTS` + `LAMBDA_NETWORK_LOGS` protection plans; a `guardduty_finding_metric` Lambda (EventBridge rule, severity ≥ 9) emits `security.guardduty.critical_finding`; MI alarm routes to `alerts_critical_topic_arn`.
- **Results:** `infra/guardduty.tf` + Lambda handler added; not yet applied to any environment — pending plan review and go-ahead per the standing cost rule.

### AD-140 — Bedrock Automated Reasoning Checks for Award Decisions (REQ-S606)
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** REQ-S606 specified Automated Reasoning on Award & Comms outputs, but the award decision has no live LLM call to attach a check to (Node 5 is deterministic; Node 7 uses f-string templates), and the guardrail is fleet-wide with no per-agent override.
- **Alternatives:** Second award-comms-only guardrail (rejected: doubles cost, needs new injection path); live `ApplyGuardrail` inside `node_award_comms` on the decision text (deferred, then later built differently); extract rules from prose via INGEST_CONTENT (rejected: no prose doc exists — the Python code is the source of truth).
- **Trade-offs:** A real formalization of the actual policy (`check_governance_policy`'s six checks) — but attached fleet-wide (rule semantics, not resource isolation, achieve scoping) and deployed out-of-band via boto3, invisible to Terraform drift detection.
- **Decision:** Hand-author `security/automated_reasoning_policy.json` from the six governance checks via IMPORT_POLICY; deploy out-of-band with `scripts/deploy_automated_reasoning_policy.py`; alarm `automated_reasoning_failed` on any `invalid`/`impossible` AR finding.
- **Results:** Live in dev 2026-08-14 (PR #289, five script bugs fixed); 2026-08-15 update: `ApplyGuardrail` now on Nodes 5/7 asserting the same facts deterministically — defense-in-depth, not stronger; 2026-08-16 update (PR #319): Bedrock rejects streaming with an AR guardrail, so the fleet guardrail split into the AR-carrying node guardrail and an AR-free `agent_path` twin built from the same content policy. Re-verified live via boto3 2026-08-19 (impl PR #331): `automatedReasoningPolicy` is present on guardrail version 3, both node5/node7 Lambdas are pinned to `GUARDRAIL_VERSION=3`, and CloudTrail shows no CI-driven `UpdateGuardrail` since 2026-07-03 — nothing has been silently reverting the attach. The limitation this ADR already recorded on 2026-08-15 — `node_bid_evaluation.py`'s decision dict never populates `rounds_used`/`approval_status`, so two of the six rules stay inconclusive in both the deterministic check and the AR claim — is now carried in the ATLAS evidence row as well, rather than only here.

### AD-141 — REQ-S607 Descoped: Contextual Grounding Check Has No Target (REQ-S607)
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** REQ-S607 specified a grounding check on Bid Evaluation Agent responses, but AD-117 removed that agent — Node 5 is deterministic Python — so there is no Bedrock invocation in the path for the check to attach to.
- **Alternatives:** Retarget to another node's Bedrock call (rejected: inventing a new requirement under an old ID); reintroduce a Bedrock call into Node 5 to host the check (rejected: reverses AD-117's deliberate removal for no capability gain); leave "not implemented" (rejected: implies future work that is false).
- **Trade-offs:** PRD-005 stops carrying a permanently-open item — but Defense Evasion coverage continues to rely on existing Partial controls, and a future LLM Bid Evaluation would need to revive the requirement.
- **Decision:** Formally descope REQ-S607, marking it `SUPERSEDED, non-normative` per the PRD-006 §7.4 convention for requirements whose mechanism was deliberately deleted.
- **Results:** PRD-005 §10.7 retagged (spec v1.9.4); no code or infra change — status closure only.

### AD-142 — ATLAS Navigator Layer Baseline (REQ-S600, REQ-S610)
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** `security/atlas-layer.json` was described as built across three changelog entries but never existed in the repo (same fabrication mode as REQ-S608's `memory_auth.py`); REQ-S610's eval-recording was blocked on the missing file — though the eval itself (`adversarial_robustness.py`) is real and running.
- **Alternatives:** Lambda with git-push credentials to commit after every run (rejected: new security-sensitive infra disproportionate to the pass); S3 relay for each run's results (deferred as scoped follow-up); a new scheduled GitHub Actions workflow (rejected: separate CI infrastructure beyond "create the file" scope).
- **Trade-offs:** REQ-S600 genuinely verifiable — every entry cites a live control and the file documents its own fabrication history — but REQ-S610 stays Partially implemented: the metric is the live record, the file is a periodic exercise-triggered snapshot.
- **Decision:** Create `security/atlas-layer.json` with real sourced mappings for all 12 §1.3 tactics plus REQ-S605/REQ-S606; record continuous results via the existing `adversarial_robustness_score` metric, with the git-tracked file refreshed on red-team-exercise cadence (REQ-T036).
- **Results:** File created 2026-08-13; REQ-S600 → Implemented, REQ-S610 → Partially implemented (spec v1.9.7). First refresh 2026-08-19 (impl PR #331) exercised the cadence and found the failure mode the file's own rules exist to prevent: REQ-S606's row still described the pre-PR-#306 state ("policy not yet built/attached… no live LLM call today") four days after the Automated Reasoning check went live (AD-140) — evidence rows go stale silently, since nothing validates them against the account.

### AD-147 — Cedar Principal Tags Carry `tenantId`, Unblocking Per-Tenant Policy
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Cedar's principal for an M2M token is `AgentCore::OAuthUser::"<sub>"` where `sub` is the App Client id, and only `scope` was known to surface as a tag — so no policy could say anything per-tenant, and every proposal to widen Cedar's footprint rested on an unverified assumption.
- **Alternatives:** Infer from the token's claims (rejected: claim-to-tag projection is AgentCore's behaviour, not Cognito's); promote the AD-127 RFC 8693 broker cutover first (rejected: that is the remedy for the negative outcome, and costs days against a half-day probe); `cedarpy` tests alone (rejected: they assume the entity under test and cannot see the Gateway's generated schema).
- **Trade-offs:** Gained per-tenant predicates on every Cedar surface and loud provisioner failures; given up mandatory `hasTag` guard boilerplate and a status-poll on every policy deploy. `sub` is still a machine identity — segregation-of-duties still needs AD-127 or a user-token surface.
- **Decision:** `tenantId` resolves as a Cedar principal tag with the correct per-tenant value — validated on dev in `LOG_ONLY` with a three-probe A/B/control design (`hasTag` → Allow, correct value → Allow, wrong value → Deny), then reverted.
- **Results:** `scripts/manage_policy_engine.py` now polls to `ACTIVE` and exits non-zero with `statusReasons` — closing a silent no-op deploy where an invalid policy left the previous one live while Terraform saw a clean apply; pinned by `scripts/tests/test_manage_policy_engine.py`. The rule generalized within a day: PR #326's deploy failed on `CreateDelivery` racing a same-apply Transaction Search enable (~68s, eventually consistent — `depends_on` would not help), so PR #327 added the same wait to `manage_gateway_observability.py` and stopped dev-deploy.yml retrying a stale saved plan. Standing rule for AD-52 provisioners: a control-plane call that returns has not necessarily taken effect. Phase 1 shipped on this basis — `pr_ingest.cedar` live in LOG_ONLY on the ingest Gateway with the first per-tenant predicate this finding made expressible; AD-127 stays deferred.

### AD-148 — AgentCore Runtime Targets Preclude Per-Tool Cedar; Gate Coarsely Now
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Phase 2/3 of the Cedar rollout assumed the three ungated MCP runtimes could be fronted with MCP targets to get per-tool Cedar actions. Both premises were false: nothing invoked those runtimes (`_invoke_mcp` built `boto3.client("agentcore-gateway")`, not an AWS service, so all ~27 call sites raised `UnknownServiceError`), and `server_protocol = "MCP"` on a runtime does not make it eligible as a Gateway MCP target.
- **Alternatives:** Front them with MCP targets as planned (rejected: an MCP-server target needs an HTTPS endpoint URL an AgentCore Runtime does not have); re-host all three as Lambdas now (rejected as disproportionate before a coarse gate exists — but recorded as phase 3's real prerequisite); skip phase 2 since nothing calls them (rejected: they are READY and reachable by any IAM principal, and `_invoke_mcp` is now fixed); delete the dead call sites (rejected: the servers implement behaviour the Skill runtime needs).
- **Trade-offs:** Gained a real boundary in front of runtimes that had none, verified-claim tenancy for the orchestrator, and ~19 of 27 call sites made functional; given up per-tool granularity (the body stays opaque), a per-tenant M2M client per tenant, and 8 call sites still blocked on two servers that were never built.
- **Decision:** An AgentCore Runtime can only be an HTTP target, so per-tool Cedar is unreachable that way; phase 2 ships the same coarse outer gate as ingest/receiving (plus AD-147's `tenantId` clause), and phase 3 requires re-hosting as `mcp.lambda` targets. Order by what gating buys — `step_functions_orchestrator` first (2 tools, one destructive, takes `tenant_id`); `dynamodb_master_data` must NOT reuse the shared interceptor, which injects `tenant_id` unconditionally into tools that accept no such parameter.
- **Results:** `_invoke_mcp` rewritten onto real `InvokeAgentRuntime` MCP calls (8 tests, paginated ARN resolution, `McpServerNotDeployed` naming the unbuilt `s3-reader`/`dynamodb-domain`); Skill role gains scoped `InvokeAgentRuntime`; `infra/mcp_gateways.tf` + `sfn_orchestrator.cedar` stand up the first Gateway in LOG_ONLY with an `orchestrate` scope and per-tenant M2M clients; PR #329 followed with the remaining two (`tenant_mdm.cedar`, `master_data.cedar` — the latter coarse-gate-only, since it must not reuse the tenant-injecting interceptor). **Addendum 2026-08-19:** all three Gateways are now built, but the calling tree is dead, not merely incomplete — `skill_runtime/server.py` imports only `skills/test_tenant/test_tenant_skill/` and references none of the three runtimes, doing its master-data reads and workflow starts with direct boto3. `ingest_purchase_requisitions` is defined twice and the live one is in `server.py`; `mcp_clients.py` already implements what `s3-reader`/`dynamodb-domain` were meant to provide. So the gates are real against IAM-reachable runtimes but probe-only, and **no ENFORCE flip should precede a real caller** — LOG_ONLY telemetry from a probe cannot predict what production traffic would be denied. **Decided 2026-08-19: the duplicate tree is NOT deleted**, so the only remaining route to a Gateway real traffic passes through is wiring the live path (`server.py`'s direct boto3 calls become Gateway calls) alongside it — and until that lands the ENFORCE blocker stands indefinitely rather than until a cleanup.


### AD-149 — A Degrading Caller Inverts an Enforcement Flip: Alarm the Bypass Before ENFORCE
**Status:** Accepted · **Theme:** 5. Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** Two sound decisions combine into a failure neither anticipates. Cedar engines are deny-by-default on a permit conditioned on `principal.getTag("scope")` resolving (AD-39/AD-40); callers degrade rather than halt so an outage cannot break the PR→PO handoff (AD-46). If the tag fails to resolve under `ENFORCE`, Cedar denies every call, Node 7 catches the 403 and writes the PO in-process — so POs keep flowing while each bypasses the Cedar engine, the REQUEST interceptor and the JWT authorizer, leaving the system strictly weaker than in `LOG_ONLY` and externally indistinguishable from healthy.
- **Alternatives:** Alarm the Gateway's own `DenyDecisions` metric (rejected as insufficient though it exists by default: it sees the denial but not the write that follows, its meaning inverts across the flip, and a wrong URL or expired secret produces no Cedar decision at all — flat zero while every PO bypasses); make the fallback fail closed under `ENFORCE` (rejected: converts a dependency outage into a halted handoff, the trade AD-46 decided the other way); flip and review telemetry manually (rejected: the only step that creates prod's engine was the one passing `ENFORCE` — a gate fed by the action it gates); no action (rejected: one `log.warning` stood between a total authorization bypass and nobody noticing).
- **Trade-offs:** Gained detection of a bypass within one 5-minute period at critical severity, and an `ENFORCE` flip that is a bounded observable step; given up one always-on alarm on a metric that should read zero forever, a classification obligation inherited by every future degrading caller, and four pending flips now gated on instrumentation that does not exist for them yet. The alarm detects, never prevents — it converts an illusion into a known risk.
- **Decision:** No control moves to `ENFORCE` until the bypass path it can trigger is alarmed **at the caller** — a control cannot observe its own circumvention. The caller classifies its own reach failures (`auth` 401/403 vs `http`/`config`/`transport`), emits a `reason`-dimensioned metric, and alarms the authorization bucket at bypassed-control severity, not degraded-dependency severity. Composes with AD-148: a real caller *and* a bypass alarm, both, before any flip.
- **Results:** Realized in impl PR #333 — `po_delivery.py`'s `_classify_gateway_failure`, `procurement/resilience po_delivery.gateway_fallback`, and `po-delivery-gateway-auth-bypass` on `alerts_critical` (the `governance_confidentiality_violation` tier, since both mean a control was bypassed); `transport`/`config` unalarmed by AD-133's per-alarm-month economics; ten tests pin that a 403 still returns `RECEIVED`, which is why it must emit a metric. This is why AD-40 now creates prod's receiving engine in `LOG_ONLY`. Applies unchanged to the four engines from PRs #326/#328/#329, none of which has a live caller or caller-side classification yet.

### AD-150 — ABAC Targets the Live Path, Not the Probed Gateways; Re-Hosting Stays Deferred
**Status:** Accepted · **Theme:** 05 Security, Governance & Trust Boundaries · **PRD:** PRD-005

- **Problem:** REQ-S704–S706 per-request ABAC is built but inert (`ABAC_TOOL_ROLES` empty on every Gateway), and the interceptor's own docstring justified that with "the ingest Gateway fronts the skill ingest tool, whose ARNs are not tenant-partitioned" — which pointed at `tenant_mdm`/`dynamodb_master_data` as the natural first targets. Both halves are false against live code: `skill_runtime/server.py` writes `tenant_id`-keyed and `tenant_id#`-prefixed tables throughout, and the suggested candidates have no live caller at all.
- **Alternatives:** ABAC on `tenant_mdm` first (rejected: protects traffic that structurally cannot arrive); on `dynamodb_master_data` (rejected outright — no interceptor fronts it, so credentials can never reach it); treat `ingest_purchase_requisitions` as fully eligible (rejected: `auto_start_workflow`'s shared state-machine ARN has no per-tenant IAM handle); re-host `skill_runtime` as a Lambda for per-tool Cedar (rejected for this phase: bigger than the MCP re-host AD-148 already declined).
- **Trade-offs:** ABAC lands where traffic actually is and `dynamodb_master_data` is permanently ruled out, at the cost of abandoning the "smallest Gateway first" ordering and accepting honest partial coverage on tools with a shared SFN or shared S3 leg.
- **Decision:** Phase 1's `ABAC_TOOL_ROLES` covers the six tenant-partitioned tools on the live ingest/receiving Gateways, scoped to their DynamoDB legs only; `tenant_mdm_emulator`/`step_functions_orchestrator` stay valid candidates once the dead skill tree is wired live; re-hosting for per-tool Cedar stays deferred, with Cedar remaining the coarse outer gate on all five Gateways.
- **Results:** Shipped in impl PR #341 — `tenant_credentials` threaded from each eligible tool into per-call, uncached boto3 clients built from the temporary credentials, with `test_abac_tenant_credentials.py` proving they reach boto3 as the AWS identity and that the excluded legs do not receive them; one further scoping correction found while wiring: `_cancel_domain_requisition`'s `requisition_index` GSI query cannot be tenant-scoped via `LeadingKeys` (different partition key) so it stays on the execution role. Corrects the false premise in the interceptor docstring and `infra/gateway.tf`'s header, both fixed in place. **2026-08-23 (impl PR #359) — the first live exercise of the ABAC roles exposed two defects invisible at rest.** Every `AssumeRole`-with-tags closed with `AccessDenied` on `sts:TagSession` (REQ-S706): that action must be granted in the *trust policy of the assumed role*, not only in the caller's identity policy, whenever `AssumeRole` carries `Tags` — which `_tenant_credentials()` always does; a caller granting itself the ability to pass session tags is not enough for the role to accept them. Both `ingest_abac` and `receiving_abac` trust policies now allow `["sts:AssumeRole", "sts:TagSession"]`. The same change corrected a condition-key error: there is **no `sts:RequestTag`** — the global key is `aws:RequestTag`, the same `aws:` prefix as the `aws:PrincipalTag/tenant_id` conditions the ABAC policies already key off — so the original `StringLike "sts:RequestTag/tenant_id"` silently matched nothing and enforced nothing; it is now `StringLike "aws:RequestTag/tenant_id" = "*"`. Both fixed in `infra/gateway.tf`.

## 6. Reliability, Resilience & Graceful Degradation

### AD-013 — Shared `invoke_agent_runtime` Wrapper
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-002

- **Problem:** Every agent node makes an A2A call into AgentCore Runtime; resilience patterns, W3C trace propagation, and per-tenant cost attribution had to be uniform, but per-node direct calls would re-implement and drift apart.
- **Alternatives:** Direct `InvokeAgentRuntime` per node (rejected: duplicated resilience, no chokepoint); per-node bespoke wrappers (rejected: same drift, thinner disguise).
- **Trade-offs:** Uniform resilience/attribution/tracing for all six agents vs. a shared critical-path dependency — one wrapper regression affects all agents at once; accepted because uniformity must be enforceable, not conventional.
- **Decision:** All agent nodes invoke exclusively through the shared `invoke_agent_runtime` wrapper, which applies all resilience patterns, propagates `traceparent`, emits `agentcore.session_seconds`, and returns `memory_degraded`.
- **Results:** Retry/breaker/bulkhead/timeout behave identically and are config-tunable (AD-45); per-tenant cost feeds FinOps (AD-61); `memory_degraded` (AD-20) and tracing (AD-31) enforced structurally. Update 2026-08-03 (PRs #254/#255): the wrapper's unpaginated `list_agent_runtimes()` broke any runtime sorted past page 1 of 10 — the concentration risk materialized (one bug, six agents); fixed with a boto3 paginator plus four call sites.

### AD-020 — `memory_degraded` Is the Boolean OR of Two Independent Signals
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-002

- **Problem:** Two memory tiers (AgentCore Memory, Mem0) fail independently; silent degradation would let quality thresholds calibrated for full memory over-trust a degraded run.
- **Alternatives:** Track each tier's status separately (rejected: every consumer re-ORs); treat degradation as hard failure (rejected: violates AD-46's graceful degradation); latch the flag permanently (rejected: a transient blip pins degraded thresholds for the negotiation's life).
- **Trade-offs:** One flag, one read site vs. conservative over-flagging when only the less-critical tier is down and inability to tell which tier degraded — the safe direction, never under-reports.
- **Decision:** A single `Negotiation.memory_degraded` boolean = OR of two ContextVar signals (AgentCore Memory session-scope, Mem0 cross-negotiation scope); cleared only when both recover; never blocks a negotiation.
- **Results:** Quality thresholds renormalize mid-negotiation once both tiers recover; read from the AD-13 wrapper response; feeds post-completion evaluators (PRD-004) and AD-46's no-auto-award guard.

### AD-045 — Seven Core Resilience Patterns, All Config-Driven
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-006

- **Problem:** Partial failures (supplier MCP down, Bedrock throttled) must not lose state or duplicate side effects; hardcoded per-site resilience constants drift and need redeploys to retune.
- **Alternatives:** Hardcoded constants per call site (rejected: redeploys, drift); library defaults (rejected: not sized for the platform, no config-plane observability).
- **Trade-offs:** Uniform named vocabulary tunable live without shipping code vs. resilience now depending on the config plane (which is exactly why AD-48 fails fast) and discipline cost at new call sites.
- **Decision:** Seven patterns at every outbound call site — retry+jitter, circuit breaker, idempotency, bulkhead, timeout, DLQ, shared A2A wrapper — all tunable from `{env}-system-config`.
- **Results:** Composed in the AD-13 wrapper; degradation policy per dependency in AD-46; per-operation caps in AD-50. Correction 2026-08-14 (PR #293): retry exhaustion published to the DLQ but never wrote the authoritative `REQUIRES_ATTENTION` status — the missing DynamoDB `entry_trigger` write was added.

### AD-046 — Graceful Degradation Tiers; Memory Failures Never Block
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-006

- **Problem:** Without explicit tiering, each developer decides whether a failing dependency halts or degrades a negotiation; quality-enhancer outages must not halt, essential-dependency outages must not degrade.
- **Alternatives:** All failures hard (rejected: needless halts); all failures degradable (rejected: allows running on stale config, forbidden by AD-48).
- **Trade-offs:** Negotiations survive memory/Mem0 outages and MCP circuit-breaks vs. silent quality drops unless `memory_degraded` is surfaced and a no-auto-award guard blocks irreversible bad calls; dependency classes must be maintained correctly as the system grows.
- **Decision:** Per-dependency degradation tiers: memory/Mem0 failures set `memory_degraded=true` and relax thresholds but never block; supplier MCP circuit-breaks; throttling backs off; essential config fails fast (AD-48); all degradation emits `procurement/resilience` metrics.
- **Results:** Tiers table in PRD-006 §3.1; flag feeds the Node 5 no-auto-award guard. Timeout-ordering invariant added 2026-06-20: dependency timeouts must sit strictly below the executor budget (A2A read timeout 150→100s under the 120s Lambda); PR #220 widened the node budget to 240s and restored 2 retry attempts, with a `fallback_engaged` reason classifier distinguishing killed containers from genuine agent errors. Third classified call site 2026-08-19 (impl PR #333): `po_delivery.gateway_fallback` buckets gateway failures into `auth`/`http`/`config`/`transport` so a Cedar DENY stops reading like a network timeout. Two findings came with it — the Decision's stated `degradation_type` dimension has drifted, since both classified sites independently converged on `reason`; and this is the first site where availability-first degradation can cancel a *security* control rather than merely lower quality, because the fallback bypasses Cedar, the interceptor and the JWT authorizer in one step. That case is recorded as AD-149, which makes the `auth`-bucket alarm a precondition for any ENFORCE flip; failing closed instead was rejected there on this ADR's own grounds.

### AD-047 — Rule-Based Kraljic Fallback with Quadrant-Specific Rejection
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-006

- **Problem:** Node 3 routing depends on the Kraljic Classifier agent; when it fails, the negotiation stalls — but auto-proceeding on rule-based classification (~75–90% vs ~95% LLM) is dangerous on high-stakes quadrants.
- **Alternatives:** Halt on any classifier unavailability (rejected: blocks low-risk items, violates AD-46); auto-proceed for all quadrants (rejected: ~75% BOTTLENECK accuracy is too dangerous for irreversible decisions).
- **Trade-offs:** Low-risk work keeps flowing vs. accuracy drop compensated by escalating the risky fallback cases to human review; the rule-based path bypasses LLM content filtering (Cedar and partition isolation remain).
- **Decision:** Fall back to deterministic rule-based classification on the same config thresholds, with quadrant-specific rejection: NON_CRITICAL auto-accepted; LEVERAGE <2 suppliers, BOTTLENECK empty pool, STRATEGIC ≥3 risk flags or no history escalate to REQUIRES_ATTENTION.
- **Results:** Fallback decisions idempotency-keyed; rejection thresholds config-sourced. 2026-08-14 (PR #290): fallback counters gained `tenant_id` and a `kraljic_fallback_quality` alarm fires on sustained fallback usage (trigger #4 in AD-16).

### AD-050 — Global Per-Operation Bulkheads in v1.0; Per-Tenant Sharding Deferred
**Status:** Accepted (outbound mechanism removed as dead code, PR #109 — see Results) · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-006

- **Problem:** Concurrency needed bounding per operation type (spot bids, A2A calls, DDB writes, supplier MCP) so one slow dependency can't exhaust shared resources; AgentCore microVMs offer no horizontal scaling.
- **Alternatives:** Per-tenant sharding from v1.0 (rejected: complexity; AD-79 already provides fairness at admission); no bulkheads, rely on AgentCore limits (rejected: those apply per session, not per operation type).
- **Trade-offs:** Simple immediate backpressure vs. one tenant with many concurrent auctions could saturate the global pool — accepted for v1.0 and deferred to Phase 2.
- **Decision:** Global per-operation caps from config (spot bids 200/20-per-auction, A2A 10, DDB writes 50, supplier MCP 20); per-tenant sharding deferred.
- **Results:** Removed as dead code (PR #109, 2026-07-02; REQ-R300–R302 superseded): the deployed graph is one A2A call per round per node, so the semaphores never queued anything. Live backpressure is inbound admission control (AD-124) alone; any future per-operation cap must be re-derived against the actual shape.

### AD-072 — Two-Tier Memory: AgentCore Memory + Mem0, Independently Degrading
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-014

- **Problem:** AgentCore Memory is scoped to `(actor_id, session_id)` and can't answer cross-negotiation questions (supplier history, bid patterns, approver preferences) — those stubs returned `[]`, degrading strategic decisions; AD-3 deliberately left that gap.
- **Alternatives:** Force cross-negotiation facts into session memory (rejected: scope doesn't survive invocations); bespoke DynamoDB tables (rejected: heavier, no semantic search).
- **Trade-offs:** Cross-negotiation knowledge the session scope structurally can't provide vs. two stores with different scopes and failure modes plus a new external dependency (Mem0, NAT egress).
- **Decision:** Two memory tiers side by side — AgentCore Memory for session scope, Mem0 as a tool-called cross-negotiation business-knowledge layer — each degrading independently, behind `mem0_enabled` (default false).
- **Results:** Division of labour established; the OR'd degradation flag (AD-20); four integration points in AD-74. As of 2026-07-22 Mem0 is still entirely unwired; IP-1 shipped via a DynamoDB substitute (AD-129), outside this two-tier model — the design remains aspirational for all four IPs.

### AD-073 — Mem0 Scoping Maps to the Domain Model
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-014

- **Problem:** Mem0's generic scoping fields are unmapped; without a convention, memory queries could cross tenant boundaries — violating AD-38 even for enhancement data.
- **Alternatives:** IAM/partition isolation inside Mem0 (rejected: not exposed by Mem0); `run_id` alone (rejected: doesn't prevent cross-tenant queries if `user_id` is unset).
- **Trade-offs:** Tenant isolation at zero infra cost vs. the guarantee depending on every call site setting `user_id=tenant_id` — weaker than the four-layer stack for primary data, proportionally to what the store holds.
- **Decision:** Map Mem0 scope onto the domain: `user_id`=`tenant_id`, `agent_id`=agent node name, `run_id`=`negotiation_id`, `app_id`=`"buyer-team"`; the mapping table is authoritative.
- **Results:** Tenant-filtered semantic search by default; node/negotiation attribution on writes; feature-flagged behind `mem0_enabled` and integrated with AD-72's two-tier model.

### AD-074 — Four Mem0 Integration Points, All Degrade Gracefully
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-014

- **Problem:** Cross-negotiation memory could be wired anywhere; without an explicit choice of integration points, call sites would proliferate untested — and per AD-46 none may block a negotiation.
- **Alternatives:** Wire every beneficial agent (rejected: untested degradation paths, unclear ROI); only IP-1 in v1.0 (rejected: IP-2–4 can ride the same circuit-breaker infra).
- **Trade-offs:** Decision-quality gains at the highest-value points vs. silent quality variation with Mem0 health, and one shared circuit breaker (5 failures/15 min) opens for all four IPs together.
- **Decision:** Exactly four integration points — supplier relationship history (IP-1), cross-negotiation bid patterns (IP-2), approver preferences (IP-3), communication effectiveness (IP-4) — each degrading to empty + relaxed thresholds, never blocking.
- **Results:** None of the four are wired against Mem0 (`mem0_enabled` still false). IP-1 shipped 2026-07-22 as a direct DynamoDB substitute (PR #236, AD-129) with its own try/except degradation; IP-2/3/4 remain unimplemented.

### AD-123 — Readiness Gate + SIGTERM Draining Close the Fail-Open Window at Both Ends of a Replica's Life
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-006

- **Problem:** A single-session AgentCore microVM has no failover fleet; `/ping` reported Healthy the instant the port bound (before model/blueprint ready), and shutdown either dropped in-flight runs or kept advertising Healthy while draining.
- **Alternatives:** Bare liveness check (rejected: the exact gap — listening ≠ able to serve); immediate hard shutdown (rejected: drops in-flight LLM runs); external health-check sidecar (rejected: extra component, no benefit).
- **Trade-offs:** Cold/half-initialized replicas never take task traffic vs. a broken blueprint surfacing as a stuck-Unhealthy replica needing log diagnosis; 30s graceful timeout remains an unconfirmed assumption against AgentCore's SIGKILL grace.
- **Decision:** `/ping` as readiness-and-drain gate via a `_Health` object: Healthy only when `ready and not draining`; `ready` set after blueprint/model resolution + optional warmup; SIGTERM flips `draining` before uvicorn's graceful shutdown waits up to `A2A_GRACEFUL_TIMEOUT`.
- **Results:** In `buyer_agent_core/serve.py`, shared by every agent entrypoint; proven by a real-SIGTERM subprocess integration test. Update 2026-07-15 (PR #217): the signal never reached AgentCore — `/ping` always returned HTTP 200 regardless of health state (body status isn't what AgentCore reads); fixed to return 503 when not Healthy, matching AD-124's shed status. Whether AgentCore re-polls fast enough during drain remains unverified.

### AD-124 — Inbound Concurrency-Cap Shedding Is the Live Backpressure AD-50's Removed Bulkhead Was Meant to Provide
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-006

- **Problem:** AD-50's outbound bulkhead was removed as dead code, leaving nothing proactive against AgentCore's 25 TPS per-agent throttle — callers hit `ThrottlingException`, retried, and eventually DLQ'd.
- **Alternatives:** Reactive throttle-then-retry alone (rejected: piles load onto the saturated replica); token-bucket rate limiting (rejected: the resource is in-flight count of multi-second runs, not arrival rate); config-sourced cap (rejected: agent boot path deliberately reads only model configuration).
- **Trade-offs:** Saturated replicas shed fast and cheaply (503 + `Retry-After: 1`) so the caller reschedules elsewhere vs. cap being a deploy-time env var (`A2A_MAX_INFLIGHT`, default 10), and `skill_runtime` carrying an inlined copy of the middleware (Docker build-context isolation).
- **Decision:** `AdmissionControl` ASGI middleware caps each agent's in-flight task POSTs and sheds with 503 once the cap is reached; GET (agent card, `/ping`) always passes, so saturation never looks like unhealth.
- **Results:** Shipped PR #45 (2026-06-25) with 23 tests; each shed emits an OTEL counter. Settled negative 2026-07-02: a live test (15–40 concurrent invokes) got zero 503s — AgentCore's front door prevents external stacking — so the defense is offline-proven, not live-triggerable. Extension PR #210: tenant-bucketed caps (`A2A_MAX_INFLIGHT_PER_TENANT`, `"_unknown"` bucket for unparseable bodies); PR #220: body drain only when bucketing is configured, 1 MiB ceiling, executor sized to the cap.

### AD-129 — DynamoDB-Only Substitute for Mem0 IP-1, Isolated from the Agent-Failure Path
**Status:** Accepted · **Theme:** 6. Reliability, Resilience & Graceful Degradation · **PRD:** PRD-014

- **Problem:** By 2026-07-22 Mem0 was still unwired (no SaaS account, no NAT egress) while `strategic_partnership` derived supplier history from a stable hash — not a real signal; IP-1's actual need is a per-supplier win/loss tally, which needs no semantic search.
- **Alternatives:** Wait for Mem0 (rejected: no committed timeline, gap stays open); stand up Mem0 just for IP-1 (rejected: disproportionate infra for a tally); fold into the agent-failure handler (rejected: a memory hiccup is not "the agent failed" — would trigger wrongful fallback pricing/escalation).
- **Trade-offs:** A real live decision-quality signal today vs. no semantic search and a second storage pattern alongside the Mem0 design for IP-2–4; isolation is IAM-enforceable and stronger than Mem0's scoping.
- **Decision:** A `supplier-memory` DynamoDB table (`tenant_id` / `{supplier_id}#{negotiation_id}`, no TTL) records outcomes and negotiated TCO after each successful strategic negotiation; reads fall back byte-for-byte to hash values on cold start; read/write failures degrade outside `_strategic_partnership`'s failure handler.
- **Results:** In `orchestrator/node_strategy_execute.py` (PR #236, 2026-07-22), 372 tests; live table creation pending `terraform apply`. Update 2026-08-06: the write path lacked authorization until AD-137 added candidate-scope validation (PR #264) — a prompt-injected agent could otherwise write history for non-candidate suppliers.


## 7. Observability & Evaluation

### AD-029 — Four-Layer Observability (Platform / Application / Domain / FinOps)
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Agentic systems resist conventional monitoring (non-determinism, multi-hop reasoning, cost opacity, silent quality drift), and one undifferentiated metric stream cannot serve SRE, procurement, and FinOps audiences at different altitudes.
- **Alternatives:** Single unified layer (rejected: lacks business context or couples KPIs to runtime internals); two layers (rejected: conflates debugging with KPI reporting); three layers with cost in Domain (original form — rejected once cost outgrew Domain and needed its own emitter, dashboard, alarms, and PRD).
- **Trade-offs:** Gained clean per-audience dashboards that evolve independently; given up a single negotiation spanning four layers (needs AD-31 correlation), explicit instrumentation contracts, and high-cardinality cost-dimension risk.
- **Decision:** Four layers split by audience/altitude: Platform (`AWS/Bedrock-AgentCore`, Lambda/States — free, no instrumentation), Application (ADOT/OTEL spans + `procurement/resilience`), Domain (custom EMF business metrics), FinOps (`procurement/cost` incl. republished AWS-billed actuals).
- **Results:** Four role-aligned dashboards realized; Layer 1 alarm coverage was broken twice (wrong namespace, then dimension-subset keys matched zero series 2026-07-04→08-01, fixed PR #252); Layer 3 emission corrected to true EMF (PR #164/#166); FinOps promoted to Layer 4 (2026-07-20). Cross-layer correlation via AD-31; evaluators read Layer 2 traces via AD-30; **2026-08-22 (PR #352) — trace payload is a budget:** a full-chain trace measured 370.5 KB against the 500 KB cap, of which `procurement.*` attributes were 1.6 KB (0.4%) and Application Signals — defaulted on by `otel-instrument`, consumed by nothing here — was ~243 KB; turned off on the 6 nodes, comms-delivery and pr-event-router for 370.5 → 169.9 KB (-54%) with every `node.*` span, SDK subsegment, exception stack, Transaction Search and the service map kept. The disable is a **three-variable set** (`OTEL_AWS_APPLICATION_SIGNALS_ENABLED=false` + `OTEL_PYTHON_DISTRO=aws_distro` + `OTEL_PYTHON_CONFIGURATOR=aws_configurator`): the flag alone strips the AWS configurator, so the OTLP-UDP exporter swap never happens and every span posts to a dead `localhost:4318` — tracing off, not down; **2026-08-22 (PR #357)** — the budget is now measured, and the first live reading corrects the number above: a real full-chain negotiation trace came to **229.2 KB (46% of the cap)**, not 169.9 KB — not a regression, just workload variance, so treat every byte figure here as a sample of one shape. X-Ray meters neither size nor `LimitExceeded`, so an hourly `trace_size_monitor` republishes both as ordinary series (the AD-132 shape) under two asymmetric alarms — `trace-limit-exceeded` lagging/authoritative, `trace-payload-bytes-high` leading/advisory. Verified live: both series exist undimensioned and match their alarms. **Caveat:** the 50-trace hourly sample is unfiltered, and only 3 of 1,764 traces in a 6h window carried a `node.*` span — the sampled max tracks `runtime-warmer` fan-outs (~85 KB), not the traces that can breach (P0 in the backlog; narrowing is blocked by the metadata-not-annotation gap)

### AD-030 — CloudWatch Transaction Search as a Hard Prerequisite for Evaluations
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Evaluators read agent traces; without account-level CloudWatch Transaction Search enabled, all evaluations silently produce no scores and the closed-loop quality system degrades invisibly.
- **Alternatives:** Custom trace-export pipeline (rejected: duplicates an AWS-native capability); status quo (rejected: silently disables all quality control).
- **Trade-offs:** Gained zero-wiring evaluations once the toggle is set; given up a missing one-time toggle that fails silently, plus region restrictions inherited from AD-8.
- **Decision:** Transaction Search is a documented hard prerequisite enabled once per account at environment provisioning, validated at provisioning time.
- **Results:** Single well-identified preflight check; AD-32/AD-34 depend on it. 100% sampling turned out to be dev's biggest CloudWatch cost driver, so PR #235 (2026-07-21) withdraws it on `release_vpc.sh` and restores on `restore_vpc.sh` — a scoped pause-window exception, since nothing runs against a paused dev VPC.

### AD-031 — W3C `traceparent` Propagation Across All A2A Calls
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** A negotiation spans a 7-node graph and multiple A2A agents; without shared correlation context each hop produces a disconnected trace, making end-to-end reconstruction and evaluator input impossible.
- **Alternatives:** Homegrown correlation ID (rejected: loses OTEL interoperability); per-agent disconnected traces (rejected: defeats observability and evaluation goals).
- **Trade-offs:** Gained one coherent distributed trace per negotiation; given up strict W3C adherence everywhere, weakest-hop fragility, and a hard dependency on the shared invocation wrapper.
- **Decision:** Propagate W3C `traceparent` on every A2A invocation, with resilience events recorded as span events; enforcement lives in the shared `invoke_agent_runtime` wrapper (AD-13).
- **Results:** Silently dark until PR #111 (ADOT layer ARN defaulted empty — exporter missing); trace continuity also broke across the human-approval pause (fixed PR #164, see AD-120); AgentCore Runtime's own ingress span remains a third gap closed by AD-130; fourth gap 2026-08-20 (PR #338) — Node 1 had no `_otel_ctx` to extract and minted a disconnected trace instead of rooting under Step Functions' native one, fixed via an `_X_AMZN_TRACE_ID` fallback. **2026-08-22 — propagation was right, export was lossy three ways (PR #342/#343, #345-#347, #349):** a bundled `opentelemetry-api==1.32.0` in `/var/task` shadowed the ADOT layer's api and dropped *every* manual INTERNAL span (fixed by unbundling + the optimized `AWSOpenTelemetryDistroPython:30` layer, which also deleted #342's collector `indexed_attributes` override — `procurement.*` stays metadata, not an annotation); two hypothesis fixes plus an OTel 1.44 bump recovered 6 of 7 spans; the last, `node.award_comms`, was X-Ray's **500 KB per-trace cap** silently discarding everything arriving after it (`Trace.LimitExceeded=True`), blown by the a2a SDK's self-instrumentation at 24-35% of a ~300-span run — fixed by `OTEL_INSTRUMENTATION_A2A_SDK_ENABLED=false`. A correctly propagated trace can still be incomplete for packaging, layer and total-volume reasons below the propagation layer; **2026-08-22 (PR #357)** — the 500 KB cap is now watched rather than only understood: `trace.limit_exceeded` is published hourly and alarmed, so a recurrence surfaces as an alarm instead of an absent span. The indexing gap this entry records is what limits it — `GetTraceSummaries` cannot filter on `procurement.negotiation_id`, so the sample is drawn from all traffic and rarely contains a full-chain trace (AD-29)

### AD-032 — AgentCore Evaluations Primary, Three Evaluator Types
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Tone/strategy need qualitative judgment, classification/ranking need deterministic accuracy, format/policy need structural checks — one mechanism would overpay or underserve.
- **Alternatives:** Third-party eval vendor as primary (rejected: external dependency, trace egress — integration later removed from scope entirely); LLM-as-Judge for everything (rejected: overpays on structural checks); code-only for everything (rejected: cannot grade tone).
- **Trade-offs:** Gained cost contained by construction and no vendor dependency; given up medium-cost qualitative judges and vendor-style richness (agentic-metric/hallucination detection).
- **Decision:** AgentCore Evaluations as production primary with three evaluator types — LLM-as-Judge (qualitative), Ground Truth (golden datasets), Code/Lambda (structural).
- **Results:** Built up incrementally: Ground Truth gained Kraljic + Bid Ranking + Phase 5 computable mirror (PR #312); Lambda tier gained 6+ evaluators; LLM-as-Judge tier has 4 judges (raw Bedrock after Phoenix/litellm blew the 50MB zip limit, PR #127). AgentCore Evaluations genuinely used only from 2026-07-11 (Phase A/B/C, Kraljic first, Phase B now all six agents + alarmed); Negotiation Quality unscored live since PR #151. The staging-gate deadlock on synthetic calibration data was broken 2026-08-19 (impl PR #332) without waiting for AD-143's labeling run: five consecutive weekly runs (2026-07-13…2026-08-10) had failed identically on `negotiation_quality` calibration SUSPENDED (MAD 0.607) against panel "human" labels that are generated placeholders (seed 20260702, self-disclaimed as carrying no evidentiary weight) — grading a judge against invented scores measures the fixture, not the judge, and a gate that cannot pass only teaches people to route around it. SUSPENDED is now advisory *while* `transcript_panel.json` declares `synthetic_scores: true`; judge consistency still blocks unconditionally. The downgrade requires the flag present **and** true, so replacing the panel with real dual-human labels restores blocking automatically with no code change. **2026-08-29 (impl PR #397) — the 17-day-empty online-eval namespace (see the pause-window ADRs' cross-references) is root-caused: every `service_names` value was the app-level `AgentSpec.name`, not the AWS-required `"<runtime-name>.<endpoint-name>"` format** — two similar-looking strings that never intersected, independent of sampling percentage or traffic volume. Fixed for all six configs. **2026-08-29 (impl PR #407) — the follow-on: the "real `score` datapoint" left unconfirmed above was never confirmable, because `score` was never a real metric.** Both online-eval alarms queried `AVG(score)`, which AgentCore's online-eval configs have never emitted — the real metric is `Builtin.Helpfulness`. Fixed (Kraljic alarm reads the plain metric; the strategy alarm combines four configs via metric-math `AVG([bottleneck, spot, leverage, strategic])`); both now show this namespace's first-ever real datapoints (kraljic 0.555, leverage 0.591, spot 0.427, bottleneck 0.460, strategic 0.419). **2026-08-29 (impl PR #401) — Kraljic `name_only`'s ~90% baseline was reading a leaked axis, not inferring:** `tools.py`'s history/market signals derive from the true axes with only ±0.1 jitter, reliably solvable outside a narrow band; widened to ±0.25 so no single signal resolves the threshold alone. Reframes `with_axes` as a structural check, not an accuracy measure — see AD-034's matching note on its gate floors. **2026-08-29 (impl PR #400) — Negotiation Quality's near-zero real-transcript scores (the 08-12/13 re-wire above) were a rubric/text-shape mismatch:** the rubric still graded a multi-bid comparative rationale from the retired Bid Evaluation Agent, not the short strategy-justification text the current source actually produces. Re-worded to grade what the text contains (cited signals, strategy-appropriate criteria); 7 real transcripts re-scored 0.0–0.45 → 0.58–0.81, floor unchanged at 0.60. **2026-08-29 (impl PR #402, E1-8) — the quarterly human-calibration cycle these thresholds were meant to eventually tune against is not coming:** AD-143's Ground Truth pipeline is retired (user decision, REQ-O221 marked retired in PRD-004), so every judge here stays permanently uncalibrated against a human reference — the SUSPENDED-calibration advisory downgrade is the permanent state, not a bridge to a future gate.

### AD-033 — Online Evaluation on 100% of Traces, Except Communication Tone (Tiered Sampling)
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Communication Tone runs per supplier message (not per negotiation) and would dominate evaluation cost, but award/rejection notifications are legally binding and need full scrutiny.
- **Alternatives:** Flat 100% tone coverage (rejected: dominant cost, no proportionate benefit); flat sampled coverage including award/rejection (rejected: blind spot on highest-risk communications).
- **Trade-offs:** Gained sharply cut evaluation cost with full scrutiny preserved where it matters and per-tenant tunability; given up undetected regressions on sampled-out messages and added config surface.
- **Decision:** 100% coverage for award/rejection notifications and high-value negotiations (≥ $50k default), 10% deterministic-hash sampling below, with per-tenant overrides.
- **Results:** Implemented at `node_award_comms.py` (PR #127 era), moved off the synchronous award path to an async self-invoke (PR #202/#203, ~10.8s latency removed, judge failures on the async path silently drop the score). Scope limited to award/rejection — negotiation-round comms still unscored.

### AD-034 — Evaluation Score Thresholds Drive Automated Actions (Closed Loop)
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Detecting quality drift is only half — thresholds must trigger actions without human latency; compliance violations need immediate escalation; degraded memory must relax thresholds to avoid false alarms.
- **Alternatives:** Alerts only (rejected: human latency unacceptable); single global threshold (rejected: different failure modes need different consequences); no action (rejected: silent drift).
- **Trade-offs:** Gained hands-off closed-loop control with immediate compliance escalation; given up flaky-evaluator false positives, bounded detection lag, and hard 1.0 gates on compliance checks.
- **Decision:** Each evaluator bound to a minimum score + CloudWatch alarm + automated action: NQ < 0.60 → model rollback; Tone < 0.80 → disable auto-send; Rationale < 0.70 → block auto-award; Kraljic < 0.90 → invalidate semantic cache; compliance < 1.00 → halt new negotiations; confidentiality violations escalate to Compliance on a single event.
- **Results:** Long journey from "not built" (2026-07-02) → detection alarms (07-04) → all 4 alarms deleted in cost trim (PR #252) → re-added (PRs #266-269) → all five automated actions built, deployed, and E2E-verified by 2026-08-16 (PRs #300, #308, #309-#317: auto-send disable, halt, model rollback via `model/previous` snapshot + epoch, auto-award block, cache-epoch invalidation). Confidentiality escalation live on `alerts_critical` topic. Two later findings qualify that: the alarms for actions 4 and 5 sat outside the prod canary's watch list from 2026-08-16 until impl PR #332 replaced the hand-maintained list with prefix discovery (AD-56), and the `alerts_critical` topic had no confirmed subscriber until 2026-08-19 (AD-133), so the escalation was live in Terraform and inert in practice. The model-rollback action's `model/previous` row is also why prod seeding is deliberately kept out of the deploy path — `seed_test_tenant.py` would overwrite it (AD-56); **2026-08-22 (PR #355, #356) — two ways a threshold can act on a number that measures the wrong thing:** a missing `PYTHONPATH` in the harness's documented usage line killed all 20 rows on an `agent_contract` import and `gate_outcome` read the 20/20 error rate as exit 3 "infra-down skip" (correct, and deliberately unchanged — but a local path mistake is indistinguishable from a live outage; fixed in docs + a `::warning::` diagnostic), and the gate's reused `session_id=eval-{category_id}` pinned warm AgentCore microVMs still serving the pre-deploy image, so it scored the old code while the endpoint reported the new version READY (AD-152; re-scored 80.0% → 90.0%). An automated action is only as trustworthy as the freshness and provenance of the score that triggers it. **2026-08-29 (impl PR #386) — action 4 was never actually live despite the 08-16 "E2E-verified" claim above:** `llm_judge.py`'s raw `json.loads()` on Claude's fenced Converse output raised on every call, so `eval_rationale_defensibility` had zero ALARM transitions from 08-15 through 08-27 — not a quiet evaluator, an empty one, masked by `notBreaching`. Fixed by porting the forced-JSON transport the other two judges already used. **2026-08-29 (VPC restored):** no longer VPC-blocked, but still unconfirmed — no real award has landed since the fix deployed to test `rationale_defensibility` against. Same day: the quadrant-suite live run confirmed action 4's metric populating (SampleCount 2.0), the model-rollback handler's `already_matched`/`filtered_out`/wrong-alarm-ignored outcomes were live-verified via hand-built SNS events (only `rolled_back` remains untested — needs a real tier divergence), the three judges' near-duplicate Bedrock transports were deduped into `evals/judge_transport.py` (PR #395), `kraljic-accuracy-gate` now only runs on PRs touching its own dependency surface (PR #399, CI cost hygiene), and both floors it and the daily gate bind to (0.80/0.90) are flagged as needing re-measurement against the widened Kraljic jitter (AD-32's PR #401 update) before either number is trusted again.

### AD-035 — ATLAS-Specific Evaluators Run on a Schedule
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** MITRE ATLAS threats (prompt injection, context poisoning, goal hijacking, supplier favoritism skew) are invisible to per-trace evaluators; skew is statistical across negotiations, adversarial replay needs live endpoints.
- **Alternatives:** Inline per-trace adversarial checks (rejected: cost/latency prohibitive); manual red-team only (rejected: no continuous deploy-gating signal); rely on per-trace evaluators (rejected: can't see cross-negotiation patterns).
- **Trade-offs:** Gained ATLAS threat detection gating releases and a compliance SLA; given up detection latency (weekly/monthly), dataset maintenance, and release velocity coupled to evaluator health.
- **Decision:** Two scheduled evaluators: Adversarial Robustness (weekly staging/monthly prod shadow, score < 1.0 blocks deploy) and Cross-Negotiation Supplier Skew (90-day rolling window, 2σ flag, 5-business-day compliance review).
- **Results:** Built and live-validated 2026-07-03 (4 live-only defects fixed); first meaningful multi-agent score 30% blocked (harness initially only exercised Kraljic); DLQ added for async failures (PR #132). Alarm/SNS consumption of the events still not built. **2026-08-29 (impl PR #386) — the 30% score above was from a manual run, not the deployed schedule:** `dev-buyer-team-adversarial-robustness`'s 900s Lambda timeout could never fit the sequential-across-prompts suite (worst case 6000s), so `adversarial_robustness_score` has never once landed in CloudWatch since this Lambda was deployed — REQ-O215's deploy-gating behavior has been inert from day one, not from a later regression. Fixed with bounded prompt-level parallelism, a deadline guard emitting a partial score instead of nothing, and an IAM grant restoring the tuned A2A timeout and registry lookup the role had been silently failing. **2026-08-29 (VPC restored):** live end-to-end run completed inside budget, real score exactly 0.0 — `strategic_partnership` and `bottleneck_negotiation` blocked 0/60 adversarial prompts each and floor the composite regardless of the other four agents' real (partial) results, a Guardrail-attachment question for those two agents, not the fix failing. **2026-08-29 (impl PR #396) — answered: it was the harness, the same bug class as the 2026-07-03 fix, not Guardrails.** `_build_request`'s catch-all branch never set `category_id`, which `bottleneck_negotiation`/`strategic_partnership`'s Pydantic request models require with no default — every one of their 60 prompts raised a pre-model-call `ValidationError` that `check_blocked()` correctly didn't count as a block, producing a 0/60 that read as "never blocks" but meant "never parsed." Fixed; a live re-run of the full suite against the fix is open follow-up. **2026-08-29, later (impl PR #403/#404/#405, `adversarial-robustness-plan.md`) — the `blocked/total` score itself was mislabelled: a MISS collapsed a real attack success with the architecture working as designed.** PR #403 adjudicates every miss into `BLOCKED_GUARDRAIL | CONTAINED | COMPLIED | ERROR` via an LLM judge and persists all 360 rows to S3. Live result: 226/360 (62.8%) BLOCKED, 84/360 (23.3%) CONTAINED (the stack working, previously an indistinguishable MISS), **46/360 (12.8%) COMPLIED — the real attack-success rate, and the number that should be quoted going forward, not the 0.383 composite.** PR #405 alarms `adversarial_attack_success_count` (partial-run-guarded) on `evaluation_alerts`, which has had a real confirmed subscriber since 2026-08-19 — this pages a live inbox. REQ-O215's deploy-gating promise remains unimplemented (an alarm, not a CI gate); the old blocked/total score is kept as a non-gating health signal. PR #404's Guardrail topic-policy widening (52/60→58/60 bare-text ceiling) is tracked under AD-43.

### AD-113 — Phase 0 Eval Scope: Ship What's Buildable, Stub the Judge, Gate the Rest
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** PRD-004's full evaluator subsystem was unbuilt; some gaps (golden dataset, CI honesty) were buildable now, but judge consistency/calibration (O220/O221) needs an LLM judge that didn't exist.
- **Alternatives:** Build the full AgentCore Evaluations subsystem now (rejected: too large, would delay load-bearing gaps); ship nothing for O220/O221 (rejected: no interim signal); leave the docstring implying full coverage (rejected: silent aspirational-vs-actual gap).
- **Trade-offs:** Gained immediate closure of concrete gaps and a cheap offline judge-consistency signal; given up the AD-32–35 closed loop remaining target design, and a standalone non-CI-gated stub.
- **Decision:** Ship buildable gaps (200-row golden dataset, CI validator, tagging, re-baselining trigger, honest `run_all.py` docstring); stub O220/O221 with offline `evals/llm_judge.py`; explicitly gate the rest behind later decisions.
- **Results:** REQ-O219/222/223/224 shipped; O220/O221 addressed by offline stub; AD-32–35 Results updated to state actual status. Follow-on decisions picked up the deferred scope.

### AD-115 — `emit_metric` Meta-Alerting: Non-Recursive Fallback Datapoint on Publish Failure
**Status:** Accepted — fallback mechanism superseded, see corrections below · Superseded by AD-121 · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Every Layer 3 metric flows through one `emit_metric` helper; a CloudWatch failure must never break the business path, but swallow-and-log creates a silent blind spot when the whole metrics pipe is dark.
- **Alternatives:** Log-only (rejected: silently blinds all dashboards/alarms); retry with backoff (rejected: hot paths must return promptly); recursive self-report (rejected: infinite-loop risk); shared single implementation (rejected: separate Docker build contexts per AD-101).
- **Trade-offs:** Gained a dead pipe observable in CloudWatch itself and full decoupling of metrics from business logic; given up a second call that also fails under full outage, and two behaviorally-identical copies maintained by convention.
- **Decision:** `emit_metric` never raises; on failure it logs and attempts one non-recursive `put_metric_data` to `procurement/observability`/`emit_metric.failures` reporting the failure.
- **Results:** Shipped PR #136/#140, then the whole mechanism became moot when PR #164/#166 moved emission to EMF (stdout) — no remote call left to fail. The gap (watching the EMF pipe) was closed differently by AD-121's heartbeat dead-man's-switch.

### AD-120 — Correlation ID Persisted as Row Data to Bridge Trace Discontinuities Beyond A2A Calls
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Two PR→PO hops carry no live call to propagate a header on: PR ingest via DynamoDB Stream, and the human-approval pause (`waitForTaskToken` resumed by a separate API call) — traces broke exactly at PR creation and human review.
- **Alternatives:** OTEL carrier in task-token payload only (rejected: resuming caller never has the payload; no business-readable ID); key off execution ARN (rejected: doesn't exist at PR-creation, doesn't cross the Stream hop); accept disconnected traces (rejected: defeats AD-29's coherent view).
- **Trade-offs:** Gained trace survival across both gaps plus one human-readable identifier; given up two parallel correlation mechanisms to keep consistent, and a persist-at-every-pause-point convention.
- **Decision:** Persist `correlation_id` (and the OTEL carrier at the approval pause) as plain row data on the PR/negotiation rows; restore explicitly at the next invocation instead of in-band forwarding.
- **Results:** Shipped PRs #160/#161/#164; `node_approval_gate.py` is the reference implementation for durable pauses. `correlation_id` generation is demo-only (`pr_generator.py`) — real-tenant MCP ingestion (AD-77) still doesn't emit one; **2026-08-20 (PR #340) — the canonical Gateway ingest path silently dropped both fields it exists to carry:** `pr_event_router` forwarded `correlation_id` as a tool argument but `skill_runtime`'s `ingest_purchase_requisitions` had no matching parameter, and FastMCP's `ArgModelBase` inherits pydantic `extra="ignore"`, so it was discarded without error; one hop later the skill's own `_start_workflow` never injected `_otel_ctx`, so a Gateway-path ingestion started a fresh root span at Node 1 anyway. Both fixed, with docstrings on both implementations now stating explicitly that they must be kept at parity by hand — the second instance of that failure mode after AD-146's

### AD-121 — Meta-Observability Heartbeat Dead-Man's-Switch
**Status:** Accepted (pause-window mechanism superseded by AD-133 — see the 2026-08-01 update) · Superseded by AD-133 · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** After EMF removed AD-115's failure mode, nothing watched the metrics pipeline itself — a broken EMF extraction during a quiet period is indistinguishable from "nothing happened," but alarming on absent business metrics would page on legitimate idle time.
- **Alternatives:** Alarm on "no business metric" (rejected: pages on quiet periods); resurrect AD-115's fallback datapoint (rejected: no remote call left to fail); `notBreaching` like every other alarm (rejected: permanently silent); leave heartbeat running through VPC pauses (rejected: guaranteed false page).
- **Trade-offs:** Gained a dead EMF pipeline observable independent of traffic, decoupled from load; given up a 5th scheduled Lambda and a 10-minute detection floor.
- **Decision:** A 5-minute scheduled heartbeat Lambda emits `pipeline_heartbeat`; an alarm with `treat_missing_data = "breaching"` (the only one in the repo) fires on 2 missed periods.
- **Results:** Shipped PR #181 with `ok_actions` so outage end is also notified; plotted on the Platform dashboard. The disable-actions pause pairing was replaced 2026-08-01 by AD-133's delete-and-replay (disable saved $0), which also makes the alarm re-arm in INSUFFICIENT_DATA on restore; **2026-08-22 (PR #353) — the dead-man's switch died of the cause it watches for:** the heartbeat Lambda was one of nine shipping the orchestrator package without the ADOT layer and failed at module import for ~16 hours, emitting no `pipeline_heartbeat` at all; the alarm went to ALARM on missing data correctly, alongside three other dead workers (outbox drain, DLQ redrive, approval sweep) — four of five alarms in ALARM were one bug and the fifth was that bug hiding itself. Structural limit: watcher and watched share a deployment package, a Terraform block and a deploy job, so a packaging fault takes both

### AD-130 — CloudWatch Vended Log Delivery Required to Resolve AgentCore's Own Platform-Generated Spans
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** AgentCore Runtime generates its own per-invocation span upstream of the container; the container correctly extracts it as parent, but the span was never delivered into the account (~1% orphaned parents, proven via a botocore hook capturing the outgoing trace header).
- **Alternatives:** Accept the dangling parent as cosmetic (rejected: AgentCore ingress was a complete black box); suppress the ASGI auto-instrumentor (rejected: deletes rather than resolves); override propagators in the container (rejected: severs root continuity).
- **Trade-offs:** Gained complete trace lineage and visibility into AgentCore ingress; given up another AD-52-shaped provider gap (delivery resources outside Terraform state, provisioner-tracked).
- **Decision:** Enable CloudWatch vended log delivery (`logType=TRACES` → X-Ray destination) for every AgentCore Runtime/Gateway via a boto3 `terraform_data` provisioner, since neither Terraform provider models observability fields.
- **Results:** Live-verified on `dev_kraljic_classifier` (2026-07-22), formalized in impl PR #249 for all 6 runtimes; the Gateway already had it from Cedar observability work but undocumented until this ADR.

### AD-132 — Headline KPIs Computed by a Daily Rollup Lambda; Automation Rate Defined as a Per-PR Deviation Flag
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Four headline KPIs (cycle time, automation rate, error rate, cost savings) had no series in CloudWatch — rates need something to compute them; the first formulation (`1 - events/negotiations`) computed -0.33 live and four of six counters lacked `tenant_id`.
- **Alternatives:** Alarm directly on a Metrics Insights expression (rejected: 3-hour alarm window cap); fix attribution only (rejected: ratio stays unbounded); compute from DynamoDB (rejected: duplicate scans of state CloudWatch already holds); dashboard-only (rejected: nothing can alarm).
- **Trade-offs:** Gained a rate bounded by construction (AVG of a 0/1 flag) with exact per-tenant attribution and alarmable series; given up a new write on the hot terminal path, a definitional discontinuity, and once-daily refresh lag.
- **Decision:** Daily `kpi_rollup` Lambda (clone of AD-126's poller) reads raw signals via Metrics Insights `GetMetricData` and republishes `procurement/kpi` series; automation rate = `1 - AVG(kpi.deviated)` where `kpi.deviated` is a 0/1 flag emitted once per terminal PR ("completed without a human").
- **Results:** Shipped after the Lambda had failed 44/44 runs for a month on the one-MI-query-per-call limit (PR #251); MI dialect constraints recorded (no IN/OR/LIKE, GROUP BY labels unset). Merged but not yet deployed as of 2026-08-01; alarms kept through AD-133's trim.

### AD-143 — Ground Truth Labeling via a Private Workforce for Judge Calibration
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** Every LLM judge is consistency-checked but never calibrated — `transcript_panel.json` carries synthetic placeholder scores, REQ-O221 requires dual-reviewer human labels, and the staging gate is deadlocked on exactly this missing input.
- **Alternatives:** Label by spreadsheet (rejected: no schema/audit, rubric drift); label real production transcripts (rejected: confidentiality, AD-75 discipline); SageMaker built-in consolidation (rejected: discards raw per-reviewer answers); synthetic placeholders forever (rejected: judges permanently uncalibrated).
- **Trade-offs:** Gained real calibration input and disagreement triage; given up a workforce to operate (Cognito pool, out-of-band users, one-shot job names), rubric-vs-traffic calibration gap, and a human-only apply discipline.
- **Decision:** SageMaker Ground Truth over a deterministic synthetic corpus (seed 20260702), private workforce on a dedicated Cognito pool, rubric-generated task UI, two reviewers per transcript with consensus reconciled by our own script (flags > 0.10 disagreement), output overwriting the panel in place — all in an isolated Terraform root.
- **Results:** Built in impl PR #297 (2026-08-15) with three unit-tested scripts; the labeling job has not been executed yet — pipeline built and tested, not yet run against the workforce. As of 2026-08-19 it is no longer the *only* way out of the deadlock it was scoped to break: impl PR #332 made SUSPENDED calibration advisory while the panel declares `synthetic_scores: true` (AD-32), so the staging gate passes today on judge consistency alone. That lowers the urgency but not the value — the flag is fail-safe by design, and dropping it when this pipeline's real dual-reviewer labels overwrite the panel re-arms calibration blocking automatically. **2026-08-29 (impl PR #402, E1-8) — reversed: this pipeline will not be run for the foreseeable future**, a user decision re-confirmed against live AWS (0 labeling jobs) before asking. REQ-O221 is marked retired in PRD-004; the advisory calibration downgrade above is now permanent, not a stopgap. `infra/ground_truth_labeling/` and its scripts stay in place, unremoved, in case the decision is reversed.

### AD-144 — Every Scheduled Lambda Registers for an Errors Alarm via a Central Map
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** `approval_sweep` failed with AccessDenied for 12 days and `kpi_rollup` failed 44/44 runs over 30 days unnoticed — scheduled Lambdas have no synchronous caller, and business alarms reading their metrics stay green when the handler never emits.
- **Alternatives:** Hand-write one-off alarms when remembered (rejected: the exact status quo that produced both incidents); alarm on business metrics going quiet (rejected: failed handlers look identical to nothing-to-report); CI/lint check (considered, not built — maps are a review convention).
- **Trade-offs:** Gained all 9 scheduled Lambdas alarming on `AWS/Lambda Errors` (the one signal the service emits regardless of handler behavior); given up convention-not-guarantee enforcement and two maps split by owning module.
- **Decision:** Two `for_each`-driven central maps register every `schedule_expression` Lambda's `Errors` metric for one shared alarm shape (Sum ≥ 1, `notBreaching` — the opposite of AD-121, since not-having-run-yet is not a failure), notifying `evaluation-alerts`; a `moved` block preserved `approval_sweep`'s history.
- **Results:** Shipped in impl PR #320 (2026-08-16). Inert until the next `restore_vpc.sh` replay since dev alarms were cost-paused (AD-133). Closes the gap AD-132's flag could not catch, since that flag depends on the Lambda running at all; **2026-08-22 (PR #353) — first real outage caught; the alarms worked, the reading of them did not:** four registry alarms went to ALARM and stayed there for a nine-Lambda ADOT-layer import failure, but the fifth alarm in ALARM was AD-121's heartbeat, whose missing-data state normally means "telemetry may be dark" — so the cluster read as an artifact rather than four workers down. Simultaneous ALARM across unrelated scheduled workers is itself a shared-cause signal, most strongly when the heartbeat is one of them; 2026-08-22 (PR #357) the registry gains `trace-size-monitor` — its two business alarms sit under `notBreaching`, so registering the emitter's own `Errors` alarm was a precondition of shipping it, not a follow-up

### AD-153 — A Reserved Synthetic Tenant Runs Real Negotiations as an End-to-End Canary
**Status:** Accepted — cadence amended 2026-08-25: hourly → daily, alarm reshaped 2-of-3 → 1-of-1 · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** The failure signal was layer-local — AD-121's heartbeat, AD-144's alarm registry and the domain alarms each watch a single surface, but nothing asserted the whole DAG on a schedule, and with ~60 real negotiations/month a full-pipeline regression could go unnoticed for days. Task 9 of the observability plan calls for an end-to-end synthetic canary over the exact production path.
- **Alternatives:** Browser-based Synthetics walk (rejected: the pipeline's real entry is a master-PR row write, not a UI, so a walk would exercise its own Selenium path); liveness-only terminal-AWARDED assertion (rejected: a scoring regression that still completes sails through — the composite band is what catches it); DynamoDB TTL for cleanup (rejected: forces production write paths to stamp a `ttl` attribute, teaching them about synthetic tenants — the property the mailbox simulator protects); reuse the dev tenant (rejected: a no-human canary would distort `automation_rate` and flood the re-baseline counter); status quo (rejected: nothing end-to-end).
- **Trade-offs:** A real scheduled liveness + weak scoring-regression signal over the exact production path with nothing in code aware of it, at the cost of a reserved tenant swept daily by a dedicated cleanup Lambda, deterministic fixture rows in live domain tables that `terraform apply` can drift, and the naming resolved — this decision keeps the word "canary" (AD-56's is now the *observation window*, AD-131's *variant rollout*), now enforced by a `pr-checks` naming gate rather than asserted. Daily costs ~$0.60/day against hourly's ~$14.40, bought with MTTD up to ~24h.
- **Decision:** Reserve tenant `canary-synthetic` (`lambda_core.CANARY_TENANT_ID` + an `is_synthetic_tenant` predicate) and run a Lambda-backed CloudWatch Synthetics canary on a fixed cadence: a pure DynamoDB client that puts one master PR row, polls the deterministic `negotiation_id` to terminal `AWARDED`, asserts the winning award's `composite_score` in [0.93, 1.0], and self-publishes `procurement/canary canary.success = 1.0` on PASS only (the managed `CloudWatchSynthetics` `SuccessPercent` never materialized, so "no datapoint" is the failure). The alarm reads it with `treat_missing_data = "breaching"` — a paused canary alarms — and is coupled to the cadence: 2-of-3 hourly (MTTD ~3h) as shipped, 1-of-1 with `period = 86400` since the 2026-08-25 move to daily. **The period is load-bearing**: the canary publishes only on PASS, so an hourly period against a daily cadence finds no datapoint ~23h/day and alarms almost continuously. The daily schedule must be `cron(0 0 * * ? *)` — Synthetics' `rate()` tops out at `rate(1 hour)` and `UpdateCanary` rejects `rate(1 day)`. Deterministic fixtures (category/item/two suppliers/governance overlay) are Terraform table items whose ids `_assert_fixtures` cross-checks; a budget pin keeps runs on the deterministic SPOT_BID path. `kpi_rollup`'s `all_tenants` queries and the re-baseline countdown exclude synthetic tenants via the predicate (the canary is the first *scheduled* stream — ~720 runs/mo vs ~60 real — and would otherwise have fired the countdown ~13× too often). `canary_cleanup` sweeps canary rows daily from the seven no-TTL tables; dedicated M2M clients bind the tenant through AD-119's `COGNITO_CLIENT_MAP`; supplier comms go to the SES mailbox simulator; a `Pausable = true` tag lets the VPC scripts suspend it with the environment — which alone proved insufficient, see Results.
- **Results:** Shipped in impl PRs #364–#369, #370, #373–#376; live-verified 2026-08-23 (PASS, composite in band, alarm reading the self-published metric). `canary_cleanup` registered in AD-144's alarm map. **2026-08-25 (impl PRs #377–#381) — the first week of operation came due on three of this ADR's own follow-ons.** (1) The cadence↔alarm coupling was exercised exactly as written: at ~$0.60/run, 24 runs/day was the largest recurring dev observability cost, so PR #378 went daily and reshaped the alarm in the same commit; PR #380 then hit the mechanical limit (`rate(1 day)` rejected → `cron`). (2) Pause-by-tag was necessary but not sufficient, twice: `stop-canary` on an already-`STOPPED` canary raises `ConflictException`, which under `set -euo pipefail` killed `release_vpc.sh` **after its alarm sweep and before the terraform teardown** — dev left blind *and* billing for NAT/EIP/endpoints (PR #377, now state-guarded, see AD-133); and stopping does not disarm the cron, so a broad apply re-asserting `synthetics_canary.tf` would restart a daily canary mid-pause — `release_vpc.sh` now sets `rate(0 hour)` first and `infra/enable_canary_cron.sh` owns the restore that `restore_vpc.sh` delegates to (PR #381; tests in #379). (3) The naming hazard is now guarded by `docs/GLOSSARY.md` + `scripts/check_naming.py` on `pr-checks` (impl PRs #383/#384), which fails six retired spellings and deliberately does not flag the bare words this ADR and AD-34 use legitimately. Follow-ons now: the cadence↔alarm↔period coupling, the cron expression duplicated between `synthetics_canary.tf` and `enable_canary_cron.sh`, fixture-drift discipline (`.tf` literals and `_assert_fixtures` must move together), the `${ENVIRONMENT}-buyer-team-` alarm-name prefix AD-133's pause sweep depends on, and any future synthetic stream joining `is_synthetic_tenant` or contaminating the same KPIs the canary was excluded from.

### AD-154 — An Append-Only Negotiation Audit Trail, Fail-Open Across Three Runtimes
**Status:** Accepted · **Theme:** 07 Observability & Evaluation · **PRD:** PRD-004

- **Problem:** `negotiations`/`bids` rows are overwritten in place each round (no history survives), orchestrator decision logs age out at 30 days, and the only per-round rationale field is a single mutable value already load-bearing for AD-034's auto-award judges — nothing durable records negotiation decisions, tool calls, or node decisions, and the prior spec for this (`REQ-O212`) was never deployed.
- **Alternatives:** A shared fourth cross-runtime package for the emitter (rejected: recouples three deliberately independent build artifacts); fail-closed on a write failure (rejected: inverts priority — the negotiation's business outcome outranks its own record-keeping); log-only on failure with no metric (rejected: fail-open plus silent logging makes a total outage indistinguishable from a quiet one); TTL-bounded retention (rejected: every other durable accountability record here is permanent); revive `REQ-O212`'s CloudWatch-Logs-Insights export (rejected: never deployed, wrong shape for a per-negotiation/per-tenant timeline).
- **Trade-offs:** Gained a durable, tamper-resistant, cross-runtime timeline immune to in-place overwrites, with redaction/truncation and a failure alarm shipped in the same slice as the table; given up an unbounded permanent-retention cost curve, three hand-synchronized emitter copies with no shared import to enforce parity, and (in v1) any query beyond a per-negotiation `Query` — no GSI, no export API, no search inside `detail`.
- **Decision:** New table `{env}-negotiation-events` (`pk=tenant_id#negotiation_id`, `sk=ts#event_id`, no TTL), written through three independent `emit_audit_event()` copies (`lambda_core`, `buyer_agent_core`, `mcp_servers/shared`) sharing one call signature and item shape. Fail-open (`except Exception` → log + `procurement/audit audit_event.failure`, dimensioned only by `exception_class`, alarmed via Metrics Insights `FROM SCHEMA(..., exception_class)` since a zero-dimension alarm can't distinguish real failures from silence); email-redacted and truncated-after-encoding `detail` (350 KB cap, applied post-JSON to avoid a re-escaping overshoot past DynamoDB's 400 KB item limit); un-scoped events date-bucket (`tenant_id#_unscoped#<date>`) instead of hot-partitioning on a bare tenant key.
- **Results:** Shipped in impl commits `4ea104a`/`5777071` (all three runtimes; a `dynamodb_master_data` MCP tool needed a nested-tenant_id fallback). Hardening pass closed 2026-08-30 (PRs #410 test-suite noop + 333-row purge, #412 truncation fix + `audit_event.skipped` metric + free-text redaction, #413 prod-only `deletion_protection`) after a post-implementation check found the agent tier had been deployed-but-unexercised (zero rows until a direct invoke). Gap-closure pass also closed 2026-08-30: PR #414 tests the failure alarm's dimension contract bidirectionally; PR #415 resolves the `Query` tool's tenant, adds an `audit_event.written` counter, and threads `actor_identity` where available; PR #416 documents alarm stickiness (14-minute `ALARM` dwell on one sparse datapoint) directly in `alarm_description`; PR #417 adds a smoke fixture (`orchestrator/smoke_sfn_agents.py`) driving a real agent-tier event; PR #418 adds the `tenant_time_index` GSI (`tenant_id`+`ts`, `INCLUDE` projection excluding `detail`) and a `scripts/audit_query.py` CLI reader. **A durable finding, not a gap:** the Step Functions execution input carries no caller identity at all — `pr_event_router._start_workflow` sends only `{tenant_id, requisition_id}`, and its own `_delegated_gateway_token` docstring states "PR ingest never has a human in the chain"; the only human identity anywhere is the HITL approver's `approved_by`, surfacing at award time in `po_delivery.py`, after every agent decision this trail records. `actor_identity` is real plumbing in all three emitters but is **empty by design for the agent tier**, not deferred work. Live-verified 2026-08-30: 403 rows (277 `tool_call` + 88 `agent_decision` at the agent tier, actors `agent:kraljic-classifier`/`agent:leverage-auction-agent`), ~15 events/negotiation, `tenant_time_index` `ACTIVE`, failure alarm `OK`. Still unshipped: a `skipped`/`written` ratio alarm (no baseline yet), a `source_layer` dimension on the failure metric (withheld on purpose — would itself break the alarm's exact-dimension match), and content search inside `detail` (forked on a PITR-vs-Scan-exporter question).


## 8. Cost Architecture & Optimization

### AD-057 — Model Tiering by Cognitive Demand
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Without tiering every agent uses the strongest, most expensive model; a worst-case negotiation costs $1.50–$3.00 against a $0.10 average target, and high-volume templated agents don't need top-tier reasoning.
- **Alternatives:** Single model for all agents (rejected: ~3–23× per-token waste); per-negotiation dynamic selection (rejected: latency + complexity, demand classification is stable); no action (rejected: target unreachable).
- **Trade-offs:** ~13–23× per-token saving on high-volume agents and config-driven mis-tier corrections, given up against quality headroom on weaker tiers — the classification assumption must hold or agents fail live.
- **Decision:** Assign each agent an abstract tier (SimpleLLM/DefaultLLM/StrategyLLM) resolved to Nova model IDs from `{env}-system-config` by `DynamicAgentFactory`; cheapest model passing the eval-validated quality bar.
- **Results:** Live tiers after three as-built revisions (PR #211): Kraljic Classifier and Spot Bidding on DefaultLLM (Nova Lite), three strategy agents on StrategyLLM (Nova Pro), Award & Comms SimpleLLM (configured, not invoked); Bid Evaluation has no tier at all — deterministic inline scoring (AD-117).

### AD-058 — Two Supported Tier Scenarios (A: Claude, B: Nova)
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Tier labels must resolve to concrete model IDs; Nova's steeper per-token discount comes with a 5-minute cache TTL vs Claude's 1 hour, changing caching economics for sparse agents — the choice must be explicit, not a hidden constant.
- **Alternatives:** Scenario A only (rejected: forecloses a legitimate 13–23× cost reduction); Scenario B only (rejected: 5-min TTL materially lowers sparse-agent hit rates); unrestricted per-agent selection (rejected: unbounded, uncostable space).
- **Trade-offs:** Operators get full visibility into the discount-vs-cache-persistence trade, given up against having to support and test two blessed model sets.
- **Decision:** Two sanctioned scenarios — A (Claude Sonnet/Haiku, 1h TTL, ~3× gap) and B (Nova Pro/Lite/Micro, 5-min TTL, ~13–23× gap) — as the only tier mappings, selected via system-config.
- **Results:** Never realized (correction note): no scenario switch or Claude code path exists; `seed_mirror.py` hardcodes a single Nova mapping — only one model family was ever deployed.

### AD-059 — Prompt Caching for All 6 Agents
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** System prompts and tool schemas are byte-identical across invocations but re-sent at full price; caching them (~10% of input rate on reads) is the primary lever to reach <$0.10/negotiation.
- **Alternatives:** Selective caching (rejected: inconsistent assembly, misses saving); no caching (rejected: target unreachable); manual per-agent config (rejected: byte-identical-prefix invariant needs a single assembly owner).
- **Trade-offs:** ~90% off cached input tokens vs a ~1.25× write premium and 5-minute TTL limits for sparsely-invoked agents; the saving is entirely contingent on prefix purity.
- **Decision:** Enable Bedrock prompt caching for all agents on the invocation-invariant prefix (system prompt + tool schemas), owned and tested by `DynamicAgentFactory` (REQ-C004).
- **Results:** Enabled via `features.prompt_caching_enabled`. Update 2026-07-08: Nova rejects `cachePoint` on `toolConfig`, so only the system prompt was cacheable until PR #167 gated the two independently; Llama gets no caching. Update 2026-07-17 (PR #221): the `cache.hit` metric this ADR names now actually exists.

### AD-060 — Semantic Cache for Kraljic Results
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Same category + same thresholds yields a deterministic quadrant — re-running the LLM is pure waste that compounds at scale, and invalidation must be automatic when thresholds change.
- **Alternatives:** Prompt caching alone (rejected: doesn't eliminate the call); in-memory agent cache (rejected: incompatible with ephemeral sessions, no tenant isolation); longer TTL (rejected: bounds staleness poorly).
- **Trade-offs:** Eliminates redundant Kraljic calls with hash-key auto-invalidation, given up against a modest saving (~$0.0008/classification — a minor lever) and a cache dependency to reason about.
- **Decision:** DynamoDB cache in `{env}-agent-session-cache` with PK `{tenant_id}#kraljic_classifier`, SK `{category_id}#{hash(profit_impact, supply_risk, thresholds)}`, 24h per-item TTL; a threshold change changes the hash, auto-invalidating.
- **Results:** Realized with `semantic_caching_enabled` flag (default true); feeds the AD-5 quadrant routing; saves ~$0.0008 per hit on top of prompt caching.

### AD-061 — Per-Tenant Cost Attribution via CUR-Joined Table
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Token-estimated cost is not billing-accurate (misses cross-region detours, tier discounts, free-tier offsets); the authoritative CUR has no `tenant_id` dimension, so the monthly per-tenant report (REQ-CST011) has no billing-accurate source.
- **Alternatives:** Estimated cost only (rejected: not invoice-grade); CUR by model without per-tenant split (rejected: no tenant dimension); real-time CUR gate (rejected: CUR lags hours — the spend gate AD-82 reads the estimated counter).
- **Trade-offs:** Billing-accurate chargeback with the CUR total preserved, given up against hours-late timeliness and deliberately maintaining two cost sources (estimated for gating, CUR for billing).
- **Decision:** Hourly EventBridge Lambda joins CUR/Athena totals with CloudWatch per-tenant token shares → attribution ratio → `{env}-tenant-cost-attribution` table (PK `tenant_id`, SK `{date_hour}#{model_id}`).
- **Results:** The pipeline was never built; the tables exist but orphaned. Updates 2026-08-16: PR #318/#319 added `tenant_id` dimensions to token/cache EMF — the attribution-ratio half is now unblocked; PR #320's per-call `cost.usd` metric gives the first real per-tenant dollar figure, but still estimated-rate-based. The CUR join and monthly report remain unbuilt; 2026-08-20 (PR #339) `context.utilization_pct` joins as a fourth per-call metric, same dual-emitter shape, with a hardcoded per-model context-window table covering only the 4 models in use; 2026-08-21 (PR #342) the `cost.usd` series finally gets a dollar-denominated alarm (`cost_spike_usd`, Metrics Insights SUM, warning-tier, first-pass $50/1h and untuned) rather than only its token-volume proxies — still the estimated source, a spike detector and not a billing control; **2026-08-23 — two cost-signal changes.** *(impl PR #361)* `agentcore.session_seconds`, which this entry's own trade-offs lean on, is no longer emitted: no alarm or dashboard consumed it and as a custom metric carrying `tenant_id` it was still billing, leaving `cost.usd` as the surviving per-call signal — the estimated-vs-CUR distinction is unchanged. *(impl PRs #362/#363)* `negotiation_cost_outlier` joins the spike detector answering a different question — "did any single negotiation cost far more than it should" — so it aggregates **`MAX`** (not `SUM`) of `negotiation.total_cost_usd` in a 1-hour window, on the one derived rather than guessed threshold in the file: 23 e2e runs measured ~$0.0025 (spot), ~$0.0024 (leverage), ~$0.0143 (bottleneck) and ~$0.0099 (strategic) per negotiation, so **$0.15 is 10× the most expensive observed strategy**. Two Metrics Insights decisions are baked in: an MI expression rather than a dimensioned alarm — CloudWatch matches an alarm's dimension set exactly, so a strategy-only alarm would have matched zero series and sat at OK forever (the PR #252 failure mode) — and a bare `FROM "procurement/cost"` rather than `FROM SCHEMA(…)`, because SCHEMA matches only series whose dimension set is exactly that list, so any emission path omitting `strategy` or `kraljic_quadrant` would drop out of the `MAX` silently, an invisible undercount and the one thing an outlier alarm must not have (PR #362 merged the SCHEMA form; PR #363's review corrected it). Both alarms remain estimated-cost and inherit every estimated-vs-CUR gap above

### AD-062 — Batch Inference for Nightly Supplier KPI Recalculation
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Supplier KPI recalculation is a large, periodic, read-only workload with no latency requirement; on-demand inference pays full price where Bedrock Batch offers 50% off.
- **Alternatives:** On-demand inference (rejected: no latency requirement exists); skip recalculation (rejected: KPIs are material to bid evaluation and awards); always-on batch (rejected: couples default deployment to batch infra without an opt-out).
- **Trade-offs:** 50% inference discount, given up against up-to-24h-stale results and batch-specific failure handling outside the request path.
- **Decision:** Nightly Bedrock Batch at 02:00 UTC behind a default-off `batch_supplier_evaluation_enabled` feature flag.
- **Results:** Never built (correction note): no batch job, flag, or schedule exists; supplier KPIs are read as static/seeded data.

### AD-092 — Cross-Family LLM-as-Judge Tier (`EvalLLM`)
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009, PRD-004

- **Problem:** Judges running on the same model family as the agents they score exhibit self-preference bias (~5–15% score inflation), degrading the signal that drives automated quality actions.
- **Alternatives:** Single Nova judge for both scenarios (rejected: same-family under Scenario B); judge on DefaultLLM (rejected: the bias itself); open-weight third-party judge (rejected: extra provider and deployment surface).
- **Trade-offs:** Unbiased judge signal, given up against an extra model family to manage — and under Scenario B, a pricier judge than the agents.
- **Decision:** A dedicated `EvalLLM` tier always set to a different family than the agents, resolved from system-config and inverting with the agent scenario.
- **Results:** The tier mechanism was never built; the cross-family property holds by a simpler route — `evals/judge_config.py::DEFAULT_EVAL_LLM_MODEL_ID` = Claude Haiku 4.5 (PR #195), a shared env-var default independent of the factory. All live agents are Nova, so the Claude judge is genuinely cross-family.

### AD-093 — Communication Generation Is O(1) in Supplier Count
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009, PRD-003

- **Problem:** Spot Bidding and Award & Comms generated one LLM communication per supplier — the dominant cost at N=10, an N-times-larger guardrail audit surface, and a latent fairness gap (the same RFQ worded differently per bidder).
- **Alternatives:** Keep per-supplier generation (rejected: wasteful, content is identical); template-only for everything (rejected: the invitation body and winner notification benefit from LLM flexibility).
- **Trade-offs:** ~$0.009 saved per negotiation and the fairness gap closed, given up against `compose_invitation_body` idempotency requirements and a single canonical body to audit.
- **Decision:** One LLM call per negotiation for the canonical invitation body with deterministic per-supplier renders; rejections from a governed zero-token template; winner-only LLM; `WinnerDisclosureGuard` switches MODIFY→REJECT.
- **Results:** Implemented in PRD-003 §2.2/§2.7; cost model updated to ~$0.035 cached / ~$0.043 on-demand; per-supplier SLA budget removed.

### AD-105 — Warm-Up Sentinel Early Exit (No LLM Cost for Keep-Alive Pings)
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Keep-alive pings flowed through the full agent pipeline to Bedrock inference with responses always discarded — ~30,000 wasted LLM calls per month across 6 agents.
- **Alternatives:** Strip the payload in the warm-up script (rejected: degenerate prompt still triggers an LLM call); per-agent prompt-builder check (rejected: 7 duplicate checks, easy to forget).
- **Trade-offs:** Zero LLM cost for warm-up pings, given up against a 3-line hot-path check and a reserved `"warmup"` key no real request may use.
- **Decision:** The shared A2A executor short-circuits `{"warmup": true}` requests immediately after extraction, before the runner/model path.
- **Results:** Implemented in `buyer_agent_core/serve.py:_make_executor`, inherited by all agents. PR #89 threads stable `runtimeSessionId` through three caller paths so the warmth maintained is actually consumed.

### AD-106 — Tool-Output Compaction via AfterToolCallEvent Hook
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Large tool results inflate every subsequent LLM call in multi-round agent loops; the session token ceiling caps total spend but not per-call growth, and prompt caching doesn't constrain the variable context.
- **Alternatives:** Orchestrator-level trimming (rejected: the tool loop is agent-internal); per-agent truncation (rejected: duplication); AgentCore Memory (rejected: shifts cost, adds complexity); LLM summarization (rejected: trades token bloat for a model call).
- **Trade-offs:** Bounded per-call token growth, zero-token and deterministic, given up against possible loss of middle detail in long fields and reserved `_compacted`/`_original_size` keys.
- **Decision:** `compact_tool_output` AfterToolCallEvent hook truncates results over 4,000 chars (head 2/3 + tail 1/3 for strings over 500 chars), registered in `AgentBlueprint.new_agent()` for all agents.
- **Results:** Implemented in `buyer_agent_core/compaction.py` with 9 tests including a prefix-purity regression; cache invariants unaffected since the hook fires post-tool.

### AD-126 — Daily Cost Explorer Poller Republishes Real AWS-Billed Cost to CloudWatch
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-009

- **Problem:** Every cost signal was a token-rate estimate with no visibility into non-Bedrock spend and no ground truth against actual billing; AD-61's CUR pipeline was never built, and nothing polled Cost Explorer at all.
- **Alternatives:** Poll hourly (rejected: ~$7/mo vs ~$0.30/mo for identical data — one call returns the whole 48h window); build AD-61's full join now (deferred: needs a tenant-dimension bridge); rely on estimates (rejected: drifts from billing).
- **Trade-offs:** First real per-service billed cost on the FinOps dashboard, given up against aggregate-only (not per-tenant) scope and up-to-24h lag.
- **Decision:** `finops_cost_poller` Lambda calls `GetCostAndUsage` (48h, hourly, grouped by SERVICE) daily via EventBridge and republishes buckets as `procurement/cost` metrics, plus an on-demand poll endpoint and dashboard widget.
- **Results:** Shipped PR #208 (merged 2026-07-13) with PR #209's suspend/resume wiring in the VPC release scripts; a narrower shipped sibling to AD-61, not a replacement. **2026-08-29 (impl PR #406) — the Decision's "hourly" was never reachable in this account: hourly Cost Explorer granularity is a per-payer opt-in this account's payer never enabled, so the Lambda had failed 100% of invocations since shipping and `aws_cost.usd` had never published a single datapoint in five weeks live.** Switched to `DAILY` granularity (date-only bounds, `lookback_days` default 2, dashboard period `3600`→`86400`); verified live (44 datapoints, $2.74 across 6 services). No `dev-buyer-team-finops-cost-poller-errors` alarm exists to have caught this — found by inspection, not paged. **2026-08-29 (impl PR #408) — DAILY is now a fallback, not hardcoded.** The poller tries `HOURLY` first and only falls back to `DAILY` on the specific opt-in `AccessDeniedException`; any other Cost Explorer error still propagates. Same outcome in this account today, but hourly resolution returns automatically if the payer account ever opts in — no redeploy needed.

### AD-133 — Delete and Replay Dev Alarms Across a Pause Window; Suppressing Alarm Actions Saves Nothing
**Status:** Accepted · **Theme:** 8. Cost Architecture & Optimization · **PRD:** PRD-004

- **Problem:** Alarms were 87% of the CloudWatch bill ($4.16/$4.77) and bill per alarm-month regardless of state or actions-enabled; 14 agent-runtime alarms matched zero series, 4 evaluation alarms paged a 0-subscriber topic, 6 duration alarms duplicated existing signals.
- **Alternatives:** Keep AD-121's disable-actions (rejected: saves $0); `terraform apply -target` on restore (rejected: drags ~75 tag rewrites + 7 runtime version cuts per cycle); leave alarms during pause (rejected: largest bill line, false breaches); alarms out of Terraform (rejected: permanent plan-visibility loss).
- **Trade-offs:** ~$4/mo pause saving plus $1.00/mo permanent from zero-coverage deletions, given up against point-in-time snapshots — alarm edits must apply while dev is up, and state history resets to INSUFFICIENT_DATA.
- **Decision:** `alarm_toggle.py` snapshots and deletes all `${ENV}-buyer-team-*` alarms (first step of release, last of restore); permanently delete the 10 zero-coverage alarms (55→45) with Terraform tombstone comments.
- **Results:** Shipped PR #252; retired AD-121's disable-actions ordering invariant; PR #253 exercised the mid-pause constraint (governance_hook_error merged-but-not-applied); PR #259 added an *opt-in* SNS email subscription. **Correction 2026-08-19 (impl PR #333):** that subscription never existed in CI. `var.alert_email` defaulted to `""` with a note to set it in a local `terraform.tfvars`, but `infra/.gitignore` ignores `*.tfvars`, so no workflow run ever saw a value — and since both subscriptions are `count = alert_email != "" ? 1 : 0`, every apply planned them *away*, destroying any subscriber applied by hand. From 2026-08-01 (the earliest date the repo records the zero-subscriber state) to 2026-08-19, every alarm in the repo — the `alerts_critical` compliance tier included — fired into the console and paged nobody, which means this ADR's "4 evaluation alarms paged a 0-subscriber topic" described the norm, not an anomaly. Fixed with a real committed default; a GitHub Actions variable would have reproduced the same bug by degrading to `""` when unset. The `count` guard stays so an environment can opt out explicitly, and AWS's confirmation email remains a one-time manual click per topic that Terraform cannot perform. **2026-08-25 (impl PR #377) — the sequencing turned an unrelated bug into a compounding one.** `release_vpc.sh` snapshot-deletes every alarm at line 32 and tears networking down at line 124; AD-153's pausable-canary loop sits between them, and its `stop-canary` on an already-`STOPPED` canary raised `ConflictException`, exiting under `set -euo pipefail` after the alarms were gone and before the teardown — dev blind *and* still billing, on an error naming neither consequence. State-guarded now, but the rule is this ADR's: **destroy-then-restore is a two-phase commit run by a shell script, so every step inserted between the phases must be idempotent or it inherits the whole blast radius.** The sweep also now carries a dependency it did not have — AD-153's canary alarm is `treat_missing_data = "breaching"` and is silenced across a pause *only* because its name starts with `${ENVIRONMENT}-buyer-team-`; renaming it out of the prefix re-arms the page on every pause. **2026-08-29 (impl PR #394, E1-3) — this ADR's own "state history does not survive the round trip" trade-off had a reproduced live consequence:** an alarm can go ALARM→forgotten across one pause cycle with nothing surfacing it (reproduced on `adversarial-robustness-errors`, 2026-08-27). Not preservable (CloudWatch can't recreate an alarm already-in-ALARM), so `alarm_toggle.py` now snapshots each alarm's pre-pause state and prints a WARNING before deleting anything in ALARM plus a NOTE after restore naming what needs a manual recheck — visibility only, no mechanics changed. **2026-08-29 (impl PR #398):** `release_vpc.sh`/`restore_vpc.sh` now hold a Terraform state-lock mutex (`tf_state_lock.sh`, built on the same S3 lock-file object Terraform's own backend uses) for their entire run, closing a race where a concurrent `dev-deploy.yml` apply or a second release/restore invocation could touch the same state mid-pause; both scripts pass `-lock=false` to their own `terraform apply` to avoid contending with themselves.


## 9. Infrastructure, Deployment & Platform Stack

### AD-007 — Standardize on Python/Strands/AgentCore/Terraform Stack with DynamoDB Dynamic Configuration
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-001

- **Problem:** The platform needed a managed agent runtime plus a way to change behavior (models, flags, thresholds) without redeploying; AWS AppConfig required a polling sidecar and added operational complexity.
- **Alternatives:** AWS AppConfig (rejected: sidecar + complexity, retired at platform v1.7.0); no standardized stack (rejected: bespoke infrastructure for deployment and behavioral governance).
- **Trade-offs:** A managed runtime removes session/scaling/isolation ops and unlocks newest platform capabilities, but carries Python 3.14/early-AgentCore maturity risk, hard AWS coupling, and incomplete Terraform provider coverage (addressed by AD-52).
- **Decision:** Standardize on Python 3.14, the Strands Agents SDK, Amazon Bedrock AgentCore (all six services), Terraform, and GitHub Actions; dynamic configuration read from one DynamoDB `{env}-system-config` table at agent instantiation via DynamicAgentFactory.
- **Results:** AgentCore provisioned across two Terraform providers plus SDK provisioners (AD-52); config reads are fail-fast with no stale cache or hardcoded fallback (AD-48), except two security-critical flags that hard-default to `false` (AD-49); the factory is the single owner of the cache-prefix invariant (AD-65).

### AD-008 — Production Region Constrained to AgentCore Evaluations Availability
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-001

- **Problem:** AgentCore Evaluations — prerequisite for the closed-loop quality control (AD-30, AD-34) — was GA in only 9 of 16 regions as of 2026-05, with higher session quotas (1,000 vs 500) in us-east-1/us-west-2 (assumption ASM-001).
- **Alternatives:** Deploy anywhere and provision Evaluations manually (rejected: unavailable in unsupported regions regardless of method); operate without Evaluations (rejected: it is the primary online quality-measurement layer whose scores drive automated actions).
- **Trade-offs:** Evaluations is available wherever the system runs and the higher quota raises the concurrency ceiling; given up is data-residency/regional flexibility and any region choice not driven by business geography.
- **Decision:** Production deploys only to a region where AgentCore Evaluations is GA, preferring us-east-1 (or us-west-2 secondary) for the 1,000-session quota.
- **Results:** All non-local environments use us-east-1 (PRD-007 §8); the quota directly informs the admission-control budget G ≈ 100 (AD-79); the five-environment model (AD-55) applies the constraint to every non-local tier.

### AD-051 — Modular Terraform with S3 Backend, DynamoDB Lock, and Per-Environment State
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** All infrastructure must be reproducible, auditable, and isolated per environment with no manual console changes (REQ-I001/I002); a single shared state would let a dev apply affect production and make concurrent applies unsafe.
- **Alternatives:** Single shared Terraform state (rejected: dev apply could corrupt staging/prod, unsafe concurrency); manual console provisioning for uncovered resources (rejected: loses auditability and reproducibility).
- **Trade-offs:** Per-environment blast-radius isolation, per-env locking, and full deploy traceability via GitSHA tags; given up is the convenience of a single state and more module I/O wiring to maintain.
- **Decision:** Modular Terraform — seven domain modules plus `iam/` and `policies/` — with state in S3 (SSE-KMS) + DynamoDB locking and separate state per environment (`dev/`, `staging/`, `prod/` key prefixes); every resource tagged `Environment`, `Project`, `ManagedBy`, `GitSHA`.
- **Results:** Module I/O table defined (PRD-007 §2); the community `terraform-aws-agentcore` module covers most AgentCore resources; this setup exposes the provider-coverage gap AD-52 addresses and the per-environment properties AD-55 governs.

### AD-052 — AgentCore Provisioned Across Two Terraform Providers with SDK Provisioners for Gaps
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** As of 2026-05 no single Terraform provider covered all AgentCore resources — Evaluator, OnlineEvaluationConfig, Policy/PolicyEngine, Dataset, and others had no Terraform resource in either provider — while REQ-I001 requires all infrastructure via Terraform.
- **Alternatives:** Wait for full provider coverage (rejected: parity not expected before the 2026-05 delivery); provision everything via SDK/CLI only (rejected: abandons plan-time visibility and state tracking for the covered set).
- **Trade-offs:** The platform is deployable today with visible, documented debt; given up is IaC purity (untracked imperative escape hatches) and simpler provider pinning across two providers.
- **Decision:** Provision AgentCore across both `hashicorp/aws` and `awscc`, filling remaining gaps with SDK provisioners (`null_resource` + AWS CLI in CI/CD) documented as explicit REQ-I001 exceptions, each with a drift-detection smoke test.
- **Results:** Evaluations and Cedar Policy are the primary CLI-provisioned exceptions; Memory is covered by `awscc`, Identity by `hashicorp/aws`; each provisioner has a drift smoke test and the exceptions are time-boxed for migration to native resources as providers mature. **2026-08-20 — the Cedar Policy carve-out has partially closed.** `aws_bedrockagentcore_policy_engine` and `aws_bedrockagentcore_policy` now exist in `hashicorp/aws` (since 6.47.0, below this repo's `>= 6.55.0` pin — no bump needed), and the two-phase ordering `manage_policy_engine.py` was built for turns out to be an ordinary same-apply Terraform dependency: the policy engine has no dependency on the Gateway, only the individual policy's statement text does, so there is no cycle. Spiked on `step_functions_orchestrator` — smallest blast radius per AD-148/AD-150 — where `terraform plan` against live dev planned 10 creates + 1 in-place update with the sentinel resolving against the real Gateway ARN; **not applied**. The observability provisioner was never an AgentCore-specific gap at all: `aws_cloudwatch_log_delivery_source`/`_destination`/`_delivery` are generic mature resources and `aws_xray_trace_segment_destination` is already natively managed in `infra/observability.tf` — the shim simply never migrated onto them. One real migration hazard surfaced: dev state's `policy_engine_configuration.arn` points at an engine the old shim script created, for which Terraform holds no state entry, so a blind apply would create a **second** engine and orphan the original — a cutover needs `terraform import` of the existing engine and policy, gateway by gateway. `dynamodb_master_data` still has no provider resource type and stays on the CLI-provisioner pattern; AgentCore Evaluations was not re-checked in this pass.

### AD-053 — AgentCore Runtime Protocol Is Immutable and Validated at Plan Time
**Status:** Accepted — mechanism corrected by AD-103 · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** Agent Runtimes are typed A2A (port 9000) and Skill Runtimes MCP (port 8000); a protocol mismatch causes silent invocation failures that are hard to diagnose after deployment.
- **Alternatives:** Discover mismatches at deploy time via integration tests (rejected: destroy-and-recreate is expensive); rely on runbook discipline (rejected: silent failures hard to attribute without tooling).
- **Trade-offs:** A hard-to-diagnose failure class is caught at plan time before resources are touched; the cost is only the policy check plus the discipline of never editing the field in place.
- **Decision:** Treat the runtime protocol as immutable and validate it at Terraform plan time via a policy check in CI/CD (REQ-I004).
- **Results:** All six agent Runtimes (AD-4) and their canary siblings (AD-131) use A2A; the skill runtime uses MCP; codified alongside the 2 GB ARM64 and 15-minute invoke ceiling as immutable platform facts. **Corrected by AD-103:** `update_agent_runtime` is full-replace and mutates the protocol in place, and the skill runtime is updated out-of-band via boto3 — a path the plan-time check never sees. **Update 2026-08-19 (impl PR #335):** the same risk also lives on the caller side, which neither AD-053 nor AD-103 guards — `warm_runtimes.py` pinged every Runtime with one A2A shape, 406ing (never warming) the four MCP servers; it now reads each Runtime's `serverProtocol` via `GetAgentRuntime` before shaping the ping.

### AD-054 — GitOps: Build-Once-Promote, Rollback as Forward Deploy
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** Deploys must be reproducible and auditable (REQ-I100–I102) with fast recovery that introduces no new variables; per-environment rebuilds risk different artifacts at each stage, violating "same bytes everywhere".
- **Alternatives:** Per-environment rebuild (rejected: breaks the same-bytes guarantee); in-place image mutation/tag overwrite (rejected: ambiguous tags, lost commit traceability).
- **Trade-offs:** The exact bytes validated in staging run in production and rollback reuses the normal deploy path with an already-built image; given up is rebuild simplicity — it depends on the ECR keep-last-10 retention policy and a known-good SHA.
- **Decision:** Git is the single source of truth: images are built once on merge to `main`, promoted unchanged dev → staging → prod with semver tags, and rollback is a roll-forward deploy to the last known-good SHA (≤ 5 min, REQ-I104).
- **Results:** `deployment.git_sha` traceability, built-once promotion, semver prod tags, ≤5-min roll-forward; feeds the observation window (AD-56) and per-component dev deploy (AD-111). Gap closed 2026-07-12: prod ECR repos now force `IMMUTABLE` (they had inherited dev's mutable default), and the promotion step was made retry-safe against `ImageTagAlreadyExistsException`. Invariant added 2026-08-19 (impl PR #333): "rollback is a forward deploy" only holds if the roll-forward apply re-passes every deploy-time `-var` the plan step passed — `receiving_policy_mode` diverged between the two, so a rollback would have changed enforcement posture along with the image tag (AD-40).

### AD-055 — Five-Environment Model with Per-Environment Cedar, PITR, and Retention Settings
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** Environments serve different purposes and risk profiles — a CI sandbox should not pay for PITR, while production must enforce Cedar (AD-40) and retain data durably; without a structured policy, lower envs are over-engineered or prod is under-protected.
- **Alternatives:** Two environments (rejected: no cost scaling for Local/CI, no staging validation step); uniform ENFORCE Cedar everywhere (rejected: would block routine dev work before policies stabilize).
- **Trade-offs:** Progressive validation with cost and safety scaled per stage; given up is parity — lower environments differ from prod (LOG_ONLY Cedar, no PITR, mock/SimpleLLM Bedrock), so prod-only behaviors are not exercised until staging.
- **Decision:** Define five environments — Local, CI, Dev, Staging, Production — each with its own Cedar enforcement mode, PITR, log retention, and observation-window configuration; Cedar is LOG_ONLY in CI/dev/staging and ENFORCE in production only; PITR is production-only.
- **Results:** Environment table (PRD-007 §8) satisfies REQ-I200–I204; the AD-8 regional constraint applies to every non-local tier; the promotion path (AD-54) and observation-window settings (AD-56) apply within these tiers.

### AD-056 — Monitoring-Only Observation Window → Per-Quadrant Smoke → 100% with Roll-Forward on Failure
**Status:** Accepted (mechanism revised at PRD-007 v1.0.33 — AgentCore has no fractional traffic routing) · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** A bad prod deploy must be caught before full rollout is confirmed, with per-path coverage across the four Kraljic quadrant workflows; AgentCore Runtimes do not support fractional traffic routing, so a true N%-traffic canary cannot be built.
- **Alternatives:** Immediate 100% deploy with post-deploy monitoring only (rejected: no confirmation gate or bounded roll-forward trigger); true fractional-traffic canary (rejected: platform does not support it); blue/green switch (rejected: two full production environments).
- **Trade-offs:** A structured confirmation gate — observation window plus per-quadrant smoke, ≤5-min roll-forward — but no blast-radius containment: a regression is live on 100% of Runtimes during the window, so detection-and-roll-forward speed, not exposure fraction, is the safety lever.
- **Decision:** The mechanism is a monitoring-only observation window: all images deploy to every Runtime at once, and rollout is *confirmed* only after an observation window (error rate, p99 latency, eval quality) passes plus one smoke negotiation per Kraljic quadrant; any failure rolls forward to the last known-good SHA within 5 minutes (`observation-window` job in `prod-deploy.yml`).
- **Results:** Satisfies REQ-I103/I104; roll-forward reuses AD-54's already-built images; `observation_window_minutes` is a per-environment override in the prod tier (AD-55); the former traffic-percentage slice has no effect on AgentCore and is not a traffic control. The retired spellings `canary-observation`, `canary_observation_minutes` and `canary_traffic_pct` are failed by the `pr-checks` naming gate (impl PR #384), so the rename cannot silently regress. **2026-08-19 (impl PR #333, and #332 for the two gates below):** neither half of the confirmation gate could have worked on a first prod run. (1) `observation-window` watched **4 of the 6** provisioned eval alarms — `eval_rationale_defensibility` and `eval_kraljic_accuracy` (AD-34 actions 4 and 5, PRs #309/#311) reached `eval_alarms.tf` but never the workflow's hand-maintained `ALARM_NAMES` list, so a regression on either passed the window unnoticed. The hand-list is replaced by prefix discovery (`$ENV-buyer-team-eval-`) plus a fail-closed `MIN_EVAL_ALARMS=6` guard, so an empty or partial result now fails the deploy instead of passing vacuously against nothing; watch set 18 → 20. The two AgentCore online-eval alarms stay out by prefix (alert-only, `notBreaching`, zero sampled sessions — AD-32), and `eval-kraljic-accuracy` has no prod emitter at all: its daily harness is pinned to the `dev` GitHub environment, so in prod it is watched for a carried-over ALARM state, not as live protection. (2) The per-quadrant smoke could not pass on a fresh environment — NON_CRITICAL was pinned to a pre-seeded dev item while the other three quadrants seeded their own; it now seeds like the rest, and a new `smoke_fixture_preflight` checks the tenant row and all four categories' suppliers up front, failing with `SMOKE FIXTURE MISSING … NOT a quadrant regression` plus the exact seed command. That distinction is the point: a smoke failure fires roll-forward, so an unseeded environment would otherwise present as a regression and trigger a rollback. Deliberately **no** auto-seed step — `seed_test_tenant.py` put_items the whole governance and model config groups, clobbering `model/default` and `model/previous`, the exact rows AD-34's model-rollback action restores from; a provisioning step that silently disarms a rollback mechanism does not belong in the deploy path, so prod seeding stays one-time and out of band.

### AD-103 — Skill-Runtime Updates Are Full-Replace; Protocol & Env Re-Asserted via a Guarded Path
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** AD-53's premises proved false as-built: `update_agent_runtime` is a full-replace call that flips `serverProtocol` in place (mutable, no recreate) and silently resets any omitted field; the skill runtime is updated out-of-band via boto3 — the exact path Terraform's plan-time check never sees — and an ad-hoc call reset it to A2A, causing every gateway invoke to 424 after ~130 s, misdiagnosed as cold-start.
- **Alternatives:** Keep AD-53's plan-time check (rejected: guards only the Terraform path the skill update does not take, and the field is mutable); move the skill runtime fully under routine `terraform apply` (rejected for now: broad applies revert the node Lambdas — the *targeted* apply is adopted instead); treat it as cold-start tuning (rejected: masked the real cause; no timeout covers a reset protocol).
- **Trade-offs:** Protocol/env resets are structurally impossible — MCP is non-overridable and the targeted apply also closes TF state drift; given up is a special-cased deploy path for the skill runtime, and full-replace semantics must now be respected for *every* runtime.
- **Decision:** Update the skill runtime only through a guarded path that re-asserts the full definition each time: `scripts/update_skill_runtime.py --tag <sha>` (hardcodes `serverProtocol=MCP`, re-passes live role/VPC config, rebuilds env to the Terraform canonical) or a targeted `terraform apply -target=...skill` with the `agent_image_tags` var; hand-writing a bare `update_agent_runtime` call is prohibited. This corrects AD-53.
- **Results:** Shipped in PR #73; live-verified — skill runtime restored to v11 READY with MCP and canonical env, post-apply plan clean, one-shot gateway `receive` returned 200/RECEIVED, gateway PO delivery on by default with the in-process boto3 fallback (AD-97) intact.

### AD-104 — Docker Image Optimization Strategy: Multi-Stage, Bytecode, Non-Root
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** All service Dockerfiles were single-stage: a 60 MB unused uv binary in the final image, package re-downloads on every build (no cache), uncompiled bytecode adding cold-start latency, root user, and no `.dockerignore` (~700 MB build context).
- **Alternatives:** Single-stage plus non-root USER only (rejected: uv bloat remains, no build/runtime separation); distroless base (rejected: no maintained Python 3.14 distroless image); `docker-bake.hcl` (deferred: not a performance bottleneck); shared cache mount across builds (rejected: BuildKit mounts are per-build scoped).
- **Trade-offs:** ~60 MB smaller runtime images, seconds-fast rebuilds, faster cold starts, non-root blast-radius reduction, near-instant build context; given up is two-stage verbosity and ~39 MB of `.pyc` (net ~21 MB smaller overall), plus cache mounts not helping on dependency-list changes.
- **Decision:** Five optimizations applied consistently: multi-stage builds (builder → runtime, `UV_LINK_MODE=copy`), BuildKit cache mounts for uv/npm/pnpm, `UV_COMPILE_BYTECODE=1`, a non-root system user (`agentuser`/`skilluser`), and a whitelist-oriented root `.dockerignore`; plus uv 0.9.3 → 0.11.25, pnpm pinned, and targeted frontend COPYs so source changes don't invalidate the install layer.
- **Results:** Implemented in PR #75 across `agent-base`, `skill_runtime`, and `test_tenant_app` (backend + frontend). AD-101 describes the what/why of the base image; this ADR is the how.

### AD-110 — Cognito Pool Custom Attributes Are Added Out-of-Band via AddCustomAttributes, Not Terraform
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** Per-user multi-tenancy needs a mutable `custom:tenantId` attribute, but AWS forbids modifying a live pool's schema via `UpdateUserPool`; Terraform's plan is a false negative (shows a safe in-place update that fails at apply), and a first attempt wrote `custom:custom:tenantId` — permanently created, unusable, unremovable.
- **Alternatives:** Add via `terraform apply` (rejected: AWS rejects the resulting `UpdateUserPool`; the "0 to destroy" plan is a mirage); destroy and recreate the pool (rejected: destroys users, app clients, and M2M bindings); rely on the plan-time signal (rejected: it reports the doomed change as safe — same false-safe class AD-103 rejected).
- **Trade-offs:** The attribute can be added to a live pool at all without destroying users; given up is IaC purity (a named REQ-I001 carve-out like AD-52) and full reproducibility — a fresh pool needs the API call re-run post-create.
- **Decision:** Provision pool custom attributes out-of-band via the native `AddCustomAttributes` API (boto3/CLI), documented as an explicit REQ-I001 exception; the TF resource carries `ignore_changes = [schema]`, with the schema block (`name = "tenantId"`, no `custom:` prefix) declaring intent only.
- **Results:** Live `custom:tenantId` (mutable) on the tenants pool; two users with distinct attributes each minted their own `tenantId` claim through the real pre-token trigger, proving per-user isolation (AD-6, AD-41); the web public client is deliberately not granted write access to the attribute (PR #98).

### AD-111 — Decoupled Per-Component Dev Deploy via Committed Tag Map
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** Dev has 9 independently deployable components; rebuilding and redeploying all of them on every merge wastes CI time and makes it impossible to ship or roll back one component without touching the other 8 (REQ-I109/I110) — AD-54 is whole-system granularity.
- **Alternatives:** Rebuild/redeploy all 9 every merge (rejected: wasted CI, coupled redeploys, worse blast radius); global `-var agent_image_tags=...` override (rejected: collapses the whole map, silently rolling back every component); staleness from the triggering push's `before..after` range (rejected: not self-healing — a failed build is retried only if a later push touches that source).
- **Trade-offs:** Single-component ship/rollback and no rebuilds of unchanged components; given up is an extra committed artifact that can desync from reality, and "what will this push deploy" depends on map state, not just the commit diff.
- **Decision:** A git-committed tag map (`infra/image_tags.auto.tfvars.json`) is the sole source of truth for which tag each component runs; the auto `dev-deploy.yml` diffs each component against its own recorded tag (self-healing staleness, `agent-base` staleness rebuilds every agent); `deploy-agent.yml` ships or rolls back exactly one component with an ECR existence check before touching Terraform state; passing `-var agent_image_tags` directly is prohibited.
- **Results:** Live-verified: correct full build+apply on merge, no-op push skipped all builds, single-component ship and rollback each < 6 min. The map-write race materialized and was closed by PR #233 (rebase + retry push of the map commit). Update 2026-08-16 (PR #325): the apply step now retries the NAT-Gateway/EIP teardown race 5× with 25 s pauses.

### AD-112 — `prevent_destroy` as the Guardrail for Dev's Automated Apply
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** Dev runs `terraform apply` automatically on every merge with no human in the loop; a plan that replaces a stateful resource would destroy and recreate it unattended, losing data (REQ-I006). No KMS key is provisioned, ruling out KMS-based protection.
- **Alternatives:** CI plan-gate requiring human review of destroys (rejected: reintroduces a manual step into auto-deploy); manual-approval gate on every apply (rejected: removes the auto-on-merge property for a rare case); KMS-based protection (rejected: adds unrelated infrastructure for something Terraform solves natively).
- **Trade-offs:** The auto-apply stays fully unattended with protection enforced by Terraform itself at plan time; given up is that the guard is a hard stop (a legitimate replace needs a deliberate, auditable code change first) and only covers resources the guard was explicitly added to.
- **Decision:** Every PRD-007-owned stateful resource — DynamoDB core tables, the `dlq_archive` and `procurement_data` S3 buckets, and the Cognito user pool — carries `lifecycle { prevent_destroy = true }`, making any destructive plan fail outright.
- **Results:** Applied across the modules plus the PRD-015 `master-data` module's 5 tables; live-verified: a forced local-only replace failed at plan time with `Error: Instance cannot be destroyed`, and the guard has never been bypassed.

### AD-114 — ADOT Layer ARN Defaults On at the Root, Not via Workflow `-var`
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-004

- **Problem:** `adot_python_layer_arn` gated the ADOT layer and X-Ray tracing but defaulted to `""` and no caller ever set it — no layer was attached and every `agentcore.invoke` span was silently dropped for an unknown period; frequent local `-target` applies rule out setting it only in a CI workflow's `-var`.
- **Alternatives:** Set the var via CI workflow `-var` only (rejected: any local `-target` apply omitting it would silently detach the layer); leave the default empty and require explicit setting (rejected: that is the fail-open status quo that produced the tracing gap).
- **Trade-offs:** Tracing is on by default on every apply path; given up is that the ARN is hardcoded to a specific AWS-managed layer version/region/arch in the variable default.
- **Decision:** Default `adot_python_layer_arn` at the root Terraform declaration to the AWS-managed us-east-1 x86_64 ADOT Python layer (`aws-otel-python-amd64-ver-1-32-0`), and wire it explicitly into every gating module, including `master-data` (pr-event-router), which had the gating logic but was never passed the var.
- **Results:** Live-verified: node Lambdas and pr-event-router show the layer attached with tracing Active; a real STRATEGIC PR→PO run produced an `agentcore.invoke` X-Ray span carrying `negotiation_id`, `agent.name`, `tenant_id`, and `model_id`, closing AD-31's gap; **2026-08-21 (PR #343) — defaulting the ARN on does not guarantee the layer's code runs:** the orchestrator build bundled `opentelemetry-api==1.32.0` into `/var/task`, which precedes `/opt/python`, splitting the tracer provider and dropping every manual span (AD-31); fixed by unbundling and moving to the optimized `AWSOpenTelemetryDistroPython:30`. **2026-08-22 (PR #353) — and it is the bill for that fix:** the unbundled api was the only thing keeping nine *layerless* functions importable, so within hours `Runtime.ImportModuleError: No module named 'opentelemetry'` hit outbox-poller (720/720 errors in 4h), dlq-redrive, observability-heartbeat and approval-sweep, with five more latent; fixed by attaching the layer, the three-variable App Signals opt-out and `tracing Active` to the scheduled workers, guarded by a programmatic no-layerless-Lambda check. A build script must not vendor what the layer provides, and a root default ARN does not make per-function attachment automatic

### AD-122 — Demo Harness and Test-Tenant App Extracted to Their Own Repository
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-013

- **Problem:** `demo-harness-project` and `test_tenant_app` lived in the platform repository sharing its CI, dependency graph, and release cadence despite no architectural coupling — widened pyright/pytest checks surfaced a real missing-import error against the demo harness, and a demo UI change could block a production deploy.
- **Alternatives:** Keep everything in one repo (rejected: shared CI and type-checking context without coupling, with concrete friction already surfacing); discard history and start fresh (rejected: throws away the evolution record `git subtree split` preserves for free); two separate repos (rejected: the two components are used together every demo run).
- **Trade-offs:** Fully isolated CI, dependencies, and release cadence; given up is shared patterns kept in sync by convention across repos, parallel copies until the platform copy is removed, and two-PR coordination for cross-repo contract changes.
- **Decision:** Extract both via `git subtree split` into an independent `buyer-team-demo` repository with its own workspace, pre-commit config, and CI; the test-tenant *data* platform (loaders, MCP servers, Test Tenant Skill — AD-75, AD-77) stays in the platform repo since production tenants exercise those code paths.
- **Results:** Shipped in PR #183; the demo repo's own pyright debt was fixed same-day in its PR #9; removal of the platform repo's `test_tenant_app` copy remains a follow-up, not yet done.

### AD-145 — `lambda_core` Is the Lambda Fleet's Shared Platform Seam
**Status:** Accepted · **Theme:** Infrastructure, Deployment & Platform Stack · **PRD:** PRD-007

- **Problem:** The same drift AD-135 fixed for the 6 LLM agents existed one layer down across the 13 Lambdas: 6+ independent `boto3.client` sites with no shared retry config, 7 copy-pasted logger blocks, 5+ hand-rolled `put_metric_data` sites — and two handlers (`ad034_halt_negotiations_remediation`, `kpi_rollup`) with zero exception handling, plus no DLQ or on-failure destination configured anywhere for any of the 13, so failures vanished after Lambda's default retries.
- **Alternatives:** Leave it as convention (rejected: the exact status quo that produced the drift, and convention had already failed silently); extend `buyer_agent_core` to cover Lambdas (rejected: materially different runtime contracts — no `/ping`/drain lifecycle, cold-start-per-invocation — and it would muddy AD-135's import-boundary guard); bundle as a zipped dependency per handler (rejected: reintroduces drift and inflates every package).
- **Trade-offs:** One place to tune retry/logging/metrics for the whole fleet, and 8 handlers that vanished failures now log, emit a dimensioned failure metric, and land in a DLQ; given up is indirection (retry config not visible inline), 8 new IAM statements, and — initially — no structural import-boundary test (closed by PR #324).
- **Decision:** A new shared package `packages/lambda_core` distributed as a real Lambda Layer (zero bundled third-party deps), attached to all 13 handlers: `aws.py` (cached clients sharing one retry `Config`), `logging_setup.py`, `metrics.py` (best-effort `put_metric`/`emit_failure_metric`), and `errors.py` with `guarded` (log + metric + re-raise, for async-invoke handlers so retries and DLQ fire) and `http_error_response` (structured error body, for the Function-URL `token_exchange_broker`). Migration was shape-specific (`pr_event_router` keeps per-record isolation; the two already-alarmed fail-closed handlers keep their hand-rolled shapes). The same effort wires a new `lambda_core_dlq` SQS queue to the 8 handlers with zero on-failure destination, granting `sqs:SendMessage` on each execution role.
- **Results:** Shipped in PR #321, merged and deployed 2026-08-16 (commit `7224317`); 1808 tests pass, ruff/pyright/terraform validate clean. PR #324 added the AST-walk boundary test (surfacing and fixing `pretoken_normaliser`/`pr_event_router` importing `boto3.dynamodb` directly). PR #325 moved the fleet's failure metrics to EMF, retiring the `http_error_response` import-time-client gotcha; `put_metrics` stays synchronous for the two backfilled-timestamp callers.

### AD-152 — A Measurement Run Mints Fresh Session IDs; Only Production Reuses Them
**Status:** Accepted · **Theme:** 09 Infrastructure, Deployment & Platform Stack · **PRD:** PRD-004

- **Problem:** AgentCore pins a `runtimeSessionId` to a warm microVM that keeps serving the image it booted with even after the endpoint rolls, while `get_agent_runtime_endpoint` reports the new version `READY` throughout. The Kraljic gate used `session_id=eval-{category_id}` — identical every run — so it silently scored the pre-deploy image, defeating REQ-O222's premise that a live-accuracy change is attributable to a spec revision. Confirmed, not inferred: `spend_materiality_index` appeared 0 times in 30 days of runtime logs while the run's tool outputs carried the field PR #354 deleted, and the two shapes were reproduced 90 seconds apart on one endpoint by varying only the session id.
- **Alternatives:** Wait out the session TTL (rejected: an unverifiable timing convention over an undocumented TTL); assert the deployed version before scoring (rejected as insufficient — the endpoint reported the new version correctly throughout); fresh ids everywhere (rejected: discards the warm reuse production depends on, AD-105 and the `load_axes` cache); status quo (rejected: attributes the old image's behaviour to the new spec, which AD-34's closed loop then acts on).
- **Trade-offs:** A gate result attributable to the image actually deployed, at the cost of a cold start per row and two conventions for one primitive whose misuse is silent in both directions.
- **Decision:** Split by intent — measurement mints a fresh per-run id (`eval-{RUN_ID}-{category_id}`), the same `--run-id` scopes span read-back in `builtin_kraljic_eval.py` (which matches *by* session id and would otherwise have matched nothing), and production keeps deterministic `kraljic-{tenant_id}-{category_id}` with the identical one-TTL staleness knowingly accepted.
- **Results:** Shipped in impl PR #356; re-scored against the same endpoint, `name_only` 16/20 = 80.0% → 18/20 = 90.0%, all four profit-axis misses resolved — so the bare-pass-at-the-floor that would have been recorded as PR #354's result (AD-5) was a measurement of the image #354 replaced. Assume the hazard applies to any post-deploy verification that talks to an AgentCore Runtime.


## 10. Capacity, Admission Control & Tenant Lifecycle

### AD-079 — Reserved + Max + Weight Capacity Model with Global Invariant
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-016

- **Problem:** Fair-sharing of the AgentCore session pool needs a per-tenant guarantee and a ceiling; the account quota (~1,000 sessions, 8–10 per negotiation) bounds total concurrency, and without a structural model one tenant's burst can starve another.
- **Alternatives:** Global ceiling only (no per-tenant floor — a low-traffic tenant could be fully blocked); fixed equal share (inflexible across different volumes); no capacity model (first high-volume tenant consumes everything).
- **Trade-offs:** Gained a guaranteed floor and a quota-derived ceiling; given up — reserved slots sit idle outside the burst pool, and `Σ reserved ≤ G` caps onboardable tenants at a given reservation level.
- **Decision:** Per-tenant `reserved_slots` (floor), `max_slots` (ceiling), `weight` (burst tie-break) in system-config, with the global invariant `Σ reserved_slots ≤ G` where `G ≈ 100 = floor(quota / sessions_per_negotiation)`.
- **Results:** Defaults 5/20/1; invariant enforced at config-write time and onboarding pre-flight; drives the three admission regions of AD-80 evaluated atomically in AD-81.

### AD-080 — Three Admission Regions; Below-Reservation Skips the Global Check
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-016

- **Problem:** An incoming negotiation is always in one of three situations relative to the tenant's own usage and the global pool; each warrants different admission behavior, and below-reservation tenants should never contend with others' bursts.
- **Alternatives:** Single global ceiling (can't distinguish reserved from burst usage); two regions without a reservation fast path (all below-max requests hit the global counter); no differentiation (burst traffic can block another tenant's reserved slots).
- **Trade-offs:** Gained contention-free latency below reservation and per-region rejection semantics; given up — burst path reads a global counter (contention latency) and the three-region logic is more complex than one check.
- **Decision:** Below reservation (`in_flight < reserved_slots`): admit, skip global check. In burst: admit only if `global_in_flight < G − Σ_other reserved_slots`. At ceiling: reject. Distinct rejection reasons (`tenant_quota_exhausted` vs `shared_pool_exhausted`) with per-region `Retry-After`.
- **Results:** Encoded in the AD-81 admission transaction (global condition expression omitted when `current_in_flight + 1 ≤ reserved_slots`); the burst region's subtraction protects every other tenant's floor.

### AD-081 — Admission Is a Single TransactWriteItems with an ACTIVE ConditionCheck
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-016

- **Problem:** Counter increments and the `DRAFT → ACTIVE` status write must never diverge (orphan counters permanently shrink capacity), and non-live tenants (PENDING/SUSPENDED/DECOMMISSIONED) must be refused atomically with the increment.
- **Alternatives:** Separate read-then-write status check (TOCTOU window); optimistic locking with separate status check (two round-trips can't be atomic); including the budget check in the transaction (not expressible as a DynamoDB condition — forced AD-82 out).
- **Trade-offs:** Gained atomicity across the whole admission path; given up — everything must be expressible as transaction conditions (the budget check isn't), and the `_global` counter is a single-partition hot spot (sharding deferred).
- **Decision:** One `TransactWriteItems`: `ConditionCheck` on `status = ACTIVE`, conditional per-tenant and global counter increments, and the negotiation `DRAFT → ACTIVE` write. Decrements fire on terminal transitions in the same transaction; `REQUIRES_ATTENTION` holds its slot.
- **Results:** Runs at Node 1 in the Orchestrator (the only component that sees every negotiation); status conditions read the AD-85 state machine; the spend gate (AD-82) runs as a non-atomic pre-check before this transaction.

### AD-082 — Spend Gate Is a Non-Atomic Pre-Check with Bounded Overshoot
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-016

- **Problem:** A preventive absolute spend cap is the only structural stop for a tenant whose abusive baseline defeats the relative anomaly alarm, but accrued cost can't be compared against config inside a DynamoDB transaction condition.
- **Alternatives:** Include the budget check in the admission transaction (not expressible as a `ConditionCheck`); use the CUR-joined table as authoritative (lags hours); route budget rejections through the capacity retry-ladder (exhausts to DLQ in ~25 minutes, long before the budget clears).
- **Trade-offs:** Gained a timely near-real-time preventive cap with distinct `Retry-After` semantics; given up exactness — a bounded overshoot of `max_slots × per-negotiation ceiling` (~$100–120 at defaults) and a small TOCTOU race.
- **Decision:** Non-atomic `Query` on the estimated accrued-cost counter before the capacity transaction; rejections set `Retry-After` to the budget-window boundary; overshoot bounded by the per-session token ceiling.
- **Results:** Reads `{env}-tenant-cost-budget` TTL buckets (no reset cron), enforces hourly-or-daily breach; the counter is deliberately not reconciled by AD-84 (fails safe by over-restricting); kill-switch ownership in AD-83 (shadow mode by default).

### AD-083 — Two Independent Kill-Switches (Concurrency vs Spend)
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-016

- **Problem:** Concurrency admission and spend enforcement have different risk profiles; spend enforcement must ship in shadow mode while concurrency stays fully active, and a single combined kill-switch would force both to share rollout risk.
- **Alternatives:** Single kill-switch (an incident in one control disables both); always enforce (a budget misconfiguration could block all tenants with no escape valve); per-tenant flags only (a system-wide bug needs a global bypass).
- **Trade-offs:** Gained independent rollout/rollback and a true global concurrency bypass for incident response; given up — two switches to track, and the spend cap protects nothing until an operator enables it.
- **Decision:** Two independent system-config flags: `tenant_admission.enforcement_enabled` (default `true`) and `tenant_budget_enforcement` (default `false` / shadow: counter maintained, metrics emitted, no blocking).
- **Results:** Stored in `{env}-system-config`; `enforcement_enabled = false` admits every request regardless of slots, letting operators recover from a broken admission transaction without a code deploy.

### AD-084 — Reconciler Lambda Corrects Counter Drift
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-016

- **Problem:** Crashes can bypass the terminal decrement, leaking counters that permanently reduce effective capacity; the concurrency counter has an observable ground truth (non-terminal negotiations) against which it can be corrected.
- **Alternatives:** Rely solely on the transactional path (leaked counters never self-correct); continuous stream-driven reconciliation (adds latency complexity for no gain); reconcile the cost-budget counter too (no ground truth — correction could flip either way).
- **Trade-offs:** Gained self-correction within ~5 minutes and an alarmed drift metric; given up — eventual consistency (drift persists up to one interval) and a 5-minute read cost over the negotiations table.
- **Decision:** A `{env}-tenant-concurrency-reconciler` Lambda on a 5-minute schedule scans non-terminal negotiations, computes observed in-flight per tenant, and corrects drift via conditional `UpdateItem`; alarms at >10% or 5 absolute drift. Only the concurrency counter is reconciled.
- **Results:** Corrections use optimistic concurrency so they can't clobber a live admission; abandoned negotiations hold their slot via the AD-16 168h `REQUIRES_ATTENTION` path until ops cancels, which fires the standard decrement.

### AD-085 — Six-State Tenant Lifecycle; PURGED Retained as a Tombstone
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-017

- **Problem:** Only the tenant create path was defined — no suspension, decommission, or purge, no authoritative `{env}-tenants` schema, and no protection against `tenant_id` reuse collisions after a tenant leaves.
- **Alternatives:** Active/deleted flag (no suspension state, no retention window, ID reuse possible); hard delete (violates audit retention); three-state ACTIVE/SUSPENDED/DELETED (conflates reversible pause with irreversible decommission, no PURGED lock on the ID).
- **Trade-offs:** Gained defined per-state admission behavior and a permanent ID reservation; given up — six states are more surface than a flag, tombstones never expire, and "deleted" is a 90-day multi-stage process.
- **Decision:** `PENDING → ACTIVE → SUSPENDED / DECOMMISSIONING → DECOMMISSIONED → PURGED`; after purge the row remains as a tombstone (only `tenant_id`, `status`, `status_transitioned_at`, `display_name`); 90-day audit window in DECOMMISSIONED.
- **Results:** Transitions recorded with reason and CloudTrail-audited; cost-attribution (AD-61) exempt from purge (own 397-day TTL); realized by the onboarding Step Function (AD-86) and suspension drain behavior (AD-87).

### AD-086 — Onboarding as a Step Function with Dry-Run Activation and Compensation
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-017

- **Problem:** Tenant onboarding spans many artifacts (Cognito, Skill Runtime, IAM, Kafka, config, counters, vault, manifests); ad hoc provisioning risks orphaned resources, and the `Σ reserved ≤ G` invariant had a runtime gap (a direct DynamoDB write bypassed the config-write check).
- **Alternatives:** Provisioning script (no idempotency, restartability, or compensation); synchronous API without dry-run (integration failures surface only in real negotiations); inline validation only (can't prove cross-component connectivity).
- **Trade-offs:** Gained seven dry-run checks proving end-to-end connectivity plus guaranteed cleanup; given up — significant machinery (Step Function, GitHub Actions bridge, onboarding-state table, 30-day async hold, ~8–15 min runtime).
- **Decision:** A 9-step idempotent, restartable Step Function: pre-flight `Σ reserved ≤ G` check, async admin hold via `WaitForCallback` (≤30 days), seven-check dry-run before `PENDING → ACTIVE`, and compensating rollback (Steps 7→2) with row deletion on terminal failure.
- **Results:** Closes the runtime invariant gap at onboarding time; failed onboarding deletes the row outright (no tombstone, per AD-85); destructive endpoints require dual-control + MFA; Cognito provisioning depends on AD-42.

### AD-087 — SUSPENDED: Reject New Admissions, Let In-Flight Drain (Optional Force-Cancel)
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-017

- **Problem:** Billing holds and compliance reviews need a reversible pause; force-stopping in-flight work is wrong for routine holds but necessary for compliance suspensions requiring immediate stop.
- **Alternatives:** Always force-cancel (destroys work-in-progress for routine holds); always drain with no force-cancel (no mechanism for legally required immediate stop); block at node boundaries (complex, still not immediate).
- **Trade-offs:** Gained default drain-to-terminal and an auditable immediate-stop option; given up — a draining suspended tenant still consumes capacity/cost, and two suspension behaviors is more API surface.
- **Decision:** On `SUSPENDED`, reject all new admissions with `tenant_suspended`; in-flight continues by default; optional `drain_in_flight = true` force-cancels non-terminal negotiations with cascading counter decrements; resume needs no dry-run.
- **Results:** Distinct rejection reason aids client handling; drain emits `tenant.drain.requested`; the AD-81 `ConditionCheck` reads the AD-85-owned `{env}-tenants` schema.

### AD-091 — `max_concurrent_bids = 200` (Reject the Proposed 200→20 Reduction)
**Status:** Accepted · **Theme:** 10 Capacity, Admission Control & Tenant Lifecycle · **PRD:** PRD-016, PRD-006

- **Problem:** A proposed reduction of `max_concurrent_bids` from 200 to 20 to limit bid fan-out cost conflated two knobs: the global throughput pool (200) and the per-auction invitation cap (`spot_bid_invitations_per_auction = 20`), the latter being the actual cost lever.
- **Alternatives:** Reduce to 20 (breaks the 200-in-90s fan-out SLA and doesn't reduce per-negotiation cost, which scales with invited-supplier count N, already capped at 20).
- **Trade-offs:** Gained — sustains the fan-out SLA (200 supports ten 20-supplier auctions; 20 would bottleneck at 13 single-supplier negotiations); given up — none, the cost concern is already met by the per-auction cap.
- **Decision:** Keep `max_concurrent_bids = 200`, sized to the Bedrock 25 TPS per-agent limit and the <90s fan-out SLA; the per-auction cap of 20 bounds per-negotiation cost.
- **Results:** Confirmed across PRD-001/003/006/016; the two-knob distinction documented at all citing sites; recorded primarily to close the open question.


## 11. Integration, Skills, Plugins & Transports

### AD-067 — One Skill + One-or-More Plugins per Tenant
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-011

- **Problem:** Every tenant integrates with different external systems over different protocols, but integration logic is largely the same; without separation, each new system duplicates logic, and external systems must never touch domain tables directly.
- **Alternatives:** One implementation per external system (maintenance scales with tenants × systems); logic in the Plugin (no reuse across transports); no action (transport and logic stay conflated, blocking multi-transport).
- **Trade-offs:** Gained logic tested once and reused across SAP/Coupa/Oracle; given up — every integration has two artifacts plus a maintained binding in per-tenant skill config.
- **Decision:** One **Skill** per tenant holds all integration logic; one-or-more **Plugins** are pure transport-declaration artifacts; the same Skill serves any ERP via `{env}-tenant-skill-config`; Plugins register as AgentCore Gateway targets.
- **Results:** Config lives in `{env}-tenant-skill-config`; SDK-transport Plugins bypass the Gateway (IAM role is the registration); the split is the prerequisite for the four-transport model (AD-68) and the bypass pattern (AD-69).

### AD-068 — Four Transports Declared in Manifest, Preference SDK > Kafka > MCP > API
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-011

- **Problem:** External systems speak SDK, Kafka, MCP, and REST; ad hoc transport choice would make the Gateway-bypass cases ungoverned, and a principled preference order is needed when a system supports several.
- **Alternatives:** Ad hoc per integration (ungoverned mix, invisible enforcement asymmetry); MCP only (excludes async volume cases, forces AWS-native through the Gateway); synchronous-first order API > MCP > Kafka > SDK (coupling over resilience).
- **Trade-offs:** Gained one integration model with the manifest as single source of truth; given up — the most-preferred transports (SDK, Kafka) skip the Gateway's Cedar and tenant-id interceptor, so each needs compensating controls.
- **Decision:** Four transports — MCP, Kafka, REST/Webhook, AWS SDK — always declared in the Plugin manifest, with preference SDK > Kafka > MCP > API; a domain may combine transports; transport is immutable at runtime.
- **Results:** Drives the AD-69 bypass behavior and which tenancy mechanism applies (Cedar/Interceptor, IAM, or topic ACLs); immutability enforced at plan time by Terraform validation (AD-53); **corrected 2026-08-20 (impl PR #340 review) — the decision is accepted but the manifest mechanism is not built:** there is no manifest schema, no `sdk_config.resource_arns` declaration, no deploy-time transport validation and no plan-time immutability check, so the "enforced at plan time" claim above does not hold; of the four transports only MCP (the two Gateways) and SDK (the Skill Runtime's IAM role) have code, Kafka has none at all — no client, topics or MSK — and REST/webhook exists only as the SES leg of supplier comms. The bypass asymmetry this ADR trades away is therefore theoretical today and becomes real the moment Kafka ships, at which point AD-69's compensating controls must land with it

### AD-069 — Kafka and SDK Transports Bypass the Gateway
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-011

- **Problem:** Kafka (persistent broker streams) and SDK (`boto3` under the Runtime IAM role) don't fit the Gateway's MCP request/response model; forcing them through would wrap natively non-MCP protocols, and the bypass must be explicit, not implicit.
- **Alternatives:** Route everything through the Gateway (MCP envelopes around non-MCP protocols); apply Cedar at the broker (not supported on Kafka); no compensating controls (violates the AD-38 defense-in-depth invariant).
- **Trade-offs:** Gained direct efficient connections with IAM-role-as-registration; given up — Gateway controls (Cedar, interceptor, per-request ABAC) don't apply to these paths, an explicit asymmetry with per-transport compensating isolation.
- **Decision:** Gateway path is MCP-only; Kafka connects directly from the Skill Runtime (MSK IAM/Confluent/SASL, topic convention `{env}.{tenant_id}.{domain}.{direction}` + broker ACLs); SDK goes via `boto3` with the Runtime IAM role; every `sdk_config.resource_arns` value must be covered by the role, validated at deploy time.
- **Results:** The bypass creates the enforcement gap closed by AD-70's predicate rewriting and AD-71's claims ceiling; SDK ARN coverage validated at deploy time (REQ-M510/REQ-I108).

### AD-070 — Tenant Predicate Rewriting + User-Action Claim Propagation
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-011

- **Problem:** Agent-generated queries (PartiQL, Athena SQL, ERP DSLs) can omit or wrong-reference `tenant_id`; Cedar can't inspect query bodies, ABAC can't stop within-resource leaks in multi-tenant tables, and the Gateway interceptor doesn't reach the bypass transports (AD-69).
- **Alternatives:** Cedar alone (tool-level, can't see query bodies); ABAC alone (no within-resource protection); prompt-based sanitization (measured 82.5% compliance vs code-level enforcement).
- **Trade-offs:** Gained a fourth tenancy layer closing the leak neither Cedar nor ABAC can reach; given up — DSL-specific query parsing per transport, which is itself a risk surface needing careful implementation.
- **Decision:** Query-bearing Plugins force-inject the JWT-bound `tenant_id` predicate before execution regardless of agent output; user-action claims propagate at the integration boundary; completes the four-layer defense (partition keys → ABAC → Cedar → predicate rewriting).
- **Results:** Rewritten predicate always sourced from the JWT-bound claim; pairs with AD-71's claim ceiling and completes AD-38's four-layer tenancy defense.

### AD-071 — `tenant_default_claims` Are the Ceiling for Per-PR Overrides
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-011

- **Problem:** Per-PR claim overrides are useful for tightening access, but a PR ingested from a compromised ERP must not be able to broaden its own required claims or introduce claim names outside the tenant's IdP.
- **Alternatives:** Allow broadening (ERP-side compromise escalates access); no per-PR overrides at all (loses legitimate tightening for sensitive PRs); validate at evaluation time (cheaper and safer to reject at the ingestion boundary).
- **Trade-offs:** Gained — an ERP compromise can only tighten authorization, never loosen it; given up — broadening requires a tenant-config change (a higher-privilege, slower operation) rather than a per-PR override.
- **Decision:** Per-PR overrides must be subset-or-equal to `tenant_default_claims` (narrowing only) and reference only `allowed_claim_namespaces` names, enforced as a deterministic code-level check at master-data ingestion; the four required default fields validated at onboarding dry-run.
- **Results:** Enforcement at ingestion (REQ-M111a/b); claim requirements evaluated at the AD-19 interrupt-resume boundary; completes the integration-boundary tenancy controls with AD-70.

### AD-097 — PO Export Decoupled from Award via a Durable Outbox
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-002 §3.7 / PRD-011 §3.2

- **Problem:** Whether Node 7 should call the export transport inline: an inline call couples award-completion latency and availability to the ERP/transport, so an outage stalls COMPLETED or forces rollback of an already-authoritative PO — a real seam for production tenants even though the test tenant's internal path hides it.
- **Alternatives:** Inline synchronous export (outage stalls or rolls back the authoritative PO); a blind poller scanning orders (can't know whether/to-whom/which-format without per-order intent).
- **Trade-offs:** Gained — award completion never blocked by ERP health, and internal/external paths share one seam; given up — a second delivery stage to build and monitor, export is eventually-consistent, and the drainer is Phase-3 work.
- **Decision:** Best-effort handoff after the authoritative `{env}-orders` write and COMPLETED transition: no configured target → the durable row is the handoff; configured target → stamp `export_status=PENDING` + target/format (the outbox) for a separate async drainer; handoff errors swallowed.
- **Results:** Realized in `orchestrator/node_award_comms.py`; order write and outbox stamp are atomic via a `TransactWriteItems` dual-write (commit_with_event); the Skill-side `export_purchase_order` drainer remains Phase-3; PR #91 fixed the missing `TransactWriteItems` IAM grant.

### AD-099 — Progressive Disclosure for MCP Skill Runtime
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports

- **Problem:** The skill runtime exposed MCP tools with no discovery mechanism — clients couldn't enumerate capabilities or read usage docs without consulting external PRD files.
- **Alternatives:** None formally evaluated.
- **Trade-offs:** Gained cheap, self-documenting discovery (L1 costs zero network calls); given up — the manifest must be kept in sync with actual tools, and out-of-sync is a documentation bug, not a runtime failure.
- **Decision:** Three-level progressive disclosure: L1 `catalog()` returns name+summary from an in-memory manifest; L2 `skill_manual(capability)` returns the co-located `SKILL.md` full text; L3 is the existing tool surface.
- **Results:** `SKILL.md` files co-located with each skill become the authoritative usage spec; unknown capabilities return structured errors; **2026-08-20 (PR #340) — L2 was dead in every deployed image since creation:** `skill_manual`'s `parent.parent / "skills"` resolved against the repo layout tests ran in, not the container `COPY` layout the image ships, so the manual directory was empty in production from day one and only L1 `catalog()` and L3 (the tools) ever worked. Verified by building a probe image mirroring the Dockerfile rather than by reasoning about the path; fixed with a `_skills_dir()` that probes both layouts

### AD-116 — FastMCP Tool Params Stay `dict`; `WithJsonSchema` Carries the Real Schema
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-013

- **Problem:** PO-receiving tools took bare `dict` params so FastMCP advertised an empty schema; typing the parameter with the Pydantic model made FastMCP's dispatch layer raise a protocol-level `ToolError` on malformed nested fields, bypassing the tool's own structured `REJECTED`/`validation_errors` contract — reproduced via direct `call_tool` against the real tool.
- **Alternatives:** Annotate with the Pydantic model (proven to break the structured-rejection contract at dispatch); bare `dict` with no schema (empty schema defeats discovery); manual validation without `WithJsonSchema` (same discoverability gap).
- **Trade-offs:** Gained a real inspectable schema at the MCP/Gateway boundary with the error contract preserved; given up — schema and runtime validation are deliberately decoupled (a "simplification" back to the model type silently reintroduces the regression) and needs a hand-rolled `_inline_defs` helper.
- **Decision:** Keep params as `dict`, attach a pre-inlined `model_json_schema()` purely for discovery via `Annotated[dict, WithJsonSchema(schema)]`; the tool body constructs the real model and maps `ValidationError` to the domain's structured rejection.
- **Results:** Applied to all three PO-receiving tools; guarded by real-dispatch tests calling `server.call_tool(...)` against actual registered tools; establishes the pattern for future FastMCP tools; complements AD-107 and AD-109.

### AD-138 — One Governed Delivery Function Replaces Six Simulated Supplier-Send Paths
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-011

- **Problem:** Six independent `_persist`/`_record_comm` paths wrote `{env}-communications` rows that never left the account, with inconsistent status casing, inconsistent `auto_send_communications` gating, and two paths lacking pre-send content-safety guards — all of which become live and irreversible once delivery turns into real SES sends.
- **Alternatives:** Wire each site to SES independently (perpetuates all six defect classes); content-safety as a mask-and-retry Bedrock Guardrail (an SES send is one-shot and irreversible — the only correct response to a scan match is not sending); skip tenant re-validation (the seam is a code-level, not data-level, guarantee).
- **Trade-offs:** Gained one place to get idempotency, tenant scoping, content safety, and the auto-send gate right, with fail-closed safety on every send; given up — a synchronous Lambda-invoke hop per send, scan false positives block rather than degrade, and `comms-delivery` becomes a single point of failure.
- **Decision:** One function `supplier_comms.delivery.deliver_communication()`: idempotent on `(negotiation_id, comm_type, supplier_id[, round_number])`, re-validates `tenant_id` against the negotiations row, fails closed on disclosure-scan matches, is the single reader of `auto_send_communications`, and sends through an injected circuit-breaker-wrapped SES call.
- **Results:** All six call sites wired and deployed (PRs #270–#279, #285); agent tools reach it only via `buyer_agent_core.comms` (synchronous Lambda invoke, AD-135 seam); auto-send remains off by default (all rows `PENDING_APPROVAL`); hardening fixes 2026-08-14 (PR #291) closed the idempotency race, added the missing `approve_and_deliver` off-ramp, normalized content matching, and added bounce handling; the 2026-08-15 build-script outage (PR #307) showed the path had been unreachable for three days — fixed; E2E run 2026-08-16 (PR #319) fixed quantity-vs-money and winner-price false positives plus a third build-script omission.

### AD-146 — `mcp_servers/shared` Becomes the MCP Servers' Platform Seam
**Status:** Accepted · **Theme:** 11 Integration, Skills, Plugins & Transports · **PRD:** PRD-013

- **Problem:** Cross-cutting concerns across the four FastMCP servers were not unified: `skill_runtime` (the only AgentCore-deployed server) imported the shared middleware directly, ran synchronous `boto3` inside async handlers on a single-worker event loop with no timeouts, and never wired `/ping` — a stalled AWS call froze every tenant's in-flight request with no way to detect it from outside.
- **Alternatives:** Fix `skill_runtime` in isolation (the next server repeats the same gaps); a dedicated wrapper for `skill_runtime` (makes the most-exposed server the exception, not the reference); fully async `aioboto3` (touches every call site's construction and error shape).
- **Trade-offs:** Gained — no single-tenant slow request can freeze the shared event loop, and factory-built servers get middleware, observability, and `/ping` by construction; given up — a thread-pool hop on every offloaded call and a one-time migration off `otel_instrumentation.py`.
- **Decision:** One factory `build_mcp_app()` in `mcp_servers/shared` wires tenant-context middleware, observability, and the health route; `skill_runtime` adopts it while offloading sync AWS calls via `asyncio.to_thread` with bounded client timeouts (10s connect / 30s read).
- **Results:** Shipped in PRs #322/#323 (2026-08-16); all four servers construct through the same entry point and `skill_runtime` gained `/ping`; third instance of the one-platform-seam-per-layer pattern (after AD-135, AD-145); PR #324 added a narrower AST-walk boundary test for the MCP layer, with `boto3` documented as a known gap; **2026-08-20 (PRs #336, #337) — the seam's structural check existed but was never wired into CI, and the servers it covers had a live bug the whole time:** three of the four `build_mcp_app` servers bound uvicorn to FastMCP's default `127.0.0.1` instead of `0.0.0.0`, so AgentCore's requests 421'd; the AST boundary test had existed since PR #324 but `mcp_servers/tests/` was absent from `pr-checks.yml`'s discovery, so nothing ran it. Both fixed together; `skill_runtime` was unaffected (AgentCore Runtime, a different hosting path)

## 12. Procurement Domain Logic

### AD-005 — Kraljic 2×2 Matrix Is the Core Routing Primitive
**Status:** Accepted · **Theme:** 12. Procurement Domain Logic · **PRD:** PRD-001

- **Problem:** Purchase requests must route to a negotiation strategy deterministically, explainably, and auditably; without a routing primitive the four-way branch is opaque and non-reproducible.
- **Alternatives:** ML-driven or free-form strategy selection (rejected: less explainable, harder to audit); no explicit primitive (rejected: no governed, repeatable branch).
- **Trade-offs:** A recognized procurement framework mapping cleanly onto a deterministic branch, given up against 2×2 simplification (boundary flips) and only four strategies — with classifier unavailability needing a 75–90%-accurate fallback.
- **Decision:** Kraljic 2×2 (`profit_impact` × `supply_risk`) classifies each item into one of four quadrants → one strategy each, thresholds (default 0.5) from `governance-policies.Kraljic_thresholds`, driving the Node 3 branch.
- **Results:** Realized as the Node 3 strategy router (AD-11); rule-based fallback with quadrant-specific escalation (AD-47); results semantically cached (AD-60); **2026-08-22 (PR #354) — the 2×2 moves out of the prompt into code:** the model was applying the matrix itself and misapplying it (two misses submitted BOTTLENECK while their own reasoning stated high profit impact), so `submit_kraljic_classification` drops its `quadrant` argument and takes `profit_impact`/`supply_risk`, with `response_builder` deriving the quadrant via a pure `classify_quadrant` that also applies the request's `classification_thresholds` deterministically instead of passing them to the model as prompt text. The profit axis was also unresolvable at its own boundary — `spend_trend` cut at spend_tier's cutoffs and left 36% of the calibration set undecidable — and is replaced by `spend_materiality_index`, the normalized-scale counterpart of the supply axis's long-reliable `supply_chain_disruption_risk`; 200-row fixture 80.7% → 85.1% (plug-in rule), live `name_only` gate 70.0% → 90.0% once AD-152 stopped it measuring the pre-deploy image. The LLM estimates the axes; code applies the matrix

### AD-009 — PR→PO Splitting: One PO per Supplier with ≥1 Awarded Item
**Status:** Accepted · **Theme:** 12. Procurement Domain Logic · **PRD:** PRD-001

- **Problem:** One PR can award items to multiple suppliers, but ERPs model supplier-scoped POs; without a splitting rule, PO assembly has no deterministic scoping and the authorization snapshot is ambiguous.
- **Alternatives:** One PO per PR (rejected: can't represent multi-supplier awards); one PO per line item (rejected: proliferation, non-standard ERP modeling).
- **Trade-offs:** Clean, deterministic, ERP-matching mapping with self-contained authorization snapshots, given up against PR→PO fan-out and partial-award bookkeeping.
- **Decision:** Group awards by `supplier_id`; each supplier with ≥1 awarded item gets exactly one ISSUED PO inheriting a claims snapshot, currency, and delivery address from the PR.
- **Results:** Realized at Node 7 — the only place PO entities are created; `supplier_index` GSI supports supplier-scoped queries over the resulting PO set.

### AD-010 — Hard Supplier Delivery Gate Before Invitation
**Status:** Accepted · **Theme:** 12. Procurement Domain Logic · **PRD:** PRD-001

- **Problem:** Inviting suppliers that cannot deliver to the PR's address or meet the delivery window wastes a full negotiation cycle and pollutes the bid pool — discovered only at bid evaluation, at maximum cost.
- **Alternatives:** Soft delivery scoring only (rejected: admits undeliverable suppliers); gate at Node 5 bid evaluation (rejected: discards at maximum cost).
- **Trade-offs:** No effort spent on undeliverable suppliers and an auditable discard set, given up against excluding marginally-over-threshold suppliers — mitigated by separating the hard `delivery_threshold_days` from the soft scoring input `delivery_ideal_days`.
- **Decision:** Hard gate at the end of Node 3: `can_deliver_to(address)` + `delivery_threshold_days`; failures recorded in `Negotiation.discarded_suppliers` with typed reasons; empty pool → CANCELLED `no_eligible_suppliers`.
- **Results:** Realized at Node 3 alongside the two-tier ESG gate (AD-18); typed discard reasons give per-condition visibility; the cancelled path is excluded from the Award Rate KPI denominator.

### AD-096 — Node 2 Kraljic Live-Schema Hybrid Short-Circuit
**Status:** Accepted · **Theme:** 12. Procurement Domain Logic · **PRD:** PRD-002

- **Problem:** Live test-tenant categories carry a seeded `kraljic_quadrant`, so the cache+agent+fallback path wasted an LLM call on a pre-determined value; categories missing axis values would also break the cache-key hash.
- **Alternatives:** Always run the full path (rejected: wastes calls, seeded quadrant is authoritative); remove the agent path for all tenants (rejected: required for real-ERP unclassified categories); seed missing axes (rejected: corrupts the source-of-truth record).
- **Trade-offs:** O(1) Node 2 cost for pre-classified categories, given up against branching on field presence (a schema rename becomes breaking) and the `0.0 → NON_CRITICAL` mapping depending on threshold defaults.
- **Decision:** Hybrid short-circuit — seeded quadrant used directly, skipping cache/agent/fallback; unclassified categories take the full path; missing axes read as 0.0 (→ NON_CRITICAL, the safest classification).
- **Results:** Shipped in PRD-002 v1.5.0, gate-free (purely data-driven); AD-60's cache and AD-47's fallback remain in effect for the unclassified path; AD-5's quadrant-to-strategy mapping unchanged.

### AD-109 — PO Receiving Lifecycle: RECEIVED Terminal State + Typed Ack/Reject + Trace Chain
**Status:** Accepted · **Theme:** 12. Procurement Domain Logic · **PRD:** PRD-013

- **Problem:** The orchestrator treated `ISSUED` as terminal, so delivered POs had no downstream state, acknowledgment/rejection had nowhere to live, and the origin trace (AD-76) was unreachable; a first attempt collected a rejection reason that was dropped in transit — write-only dead data.
- **Alternatives:** ISSUED-terminal (rejected: can't model buyer receiving); free-form status strings (rejected: unauditable, dead data); recompute the trace by scanning on read (rejected: costly and fragile vs O(1) stored trace).
- **Trade-offs:** First-class auditable receiving actions, given up against correcting the "ISSUED = done" assumption across e2e assertions and requiring every reader to carry the new fields end-to-end.
- **Decision:** A receiving lifecycle owned by the receiving domain: terminal state `RECEIVED`, typed `acknowledge`/`reject` transitions persisting timestamp + reason, and a typed `Trace` materialized on the order row.
- **Results:** Shipped PR #92/#95/#87 in the test_tenant_app backend and PO Inbox UI; picks up where AD-97's outbox handoff leaves off and materializes the AD-76 identity chain; **2026-08-20 (PR #340) — the delivered PO was missing two canonical fields since this ADR's own PR #92/#95:** `_build_received_po` never set `delivery_address` or `issued_at` (both canonical per PRD-011 §3.2/§4), so every PO reached the receiving domain with no ship-to address and no issue timestamp; fixed by threading the address through and stamping `issued_at` at assembly (idempotent on `order_id`, so a replay never overwrites the first). Both fields are optional on `IncomingPurchaseOrder`, so pre-fix POs still validate — no backfill. Same PR corrected `po_delivery.py`'s docstring, stale since PR #67, which still called the Gateway receive path off-by-default

### AD-118 — Walk-Away Price Ceiling: Orchestrator Enforces Budget on Agent-Returned Prices
**Status:** Accepted · **Theme:** 12. Procurement Domain Logic · **PRD:** PRD-002

- **Problem:** "Never exceed budget_limit" was a prompt convention, not an invariant — prompt drift or adversarial input could yield above-ceiling amounts written straight into `{env}-bids`.
- **Alternatives:** Reject the whole negotiation on any violation (rejected: too aggressive — valid bids should survive); correction round-trip to the agent (rejected: extra LLM latency/cost for a deterministic fix); silent cap at budget_limit (rejected: misrepresents the supplier's actual bid).
- **Trade-offs:** A code-enforced ceiling invariant across all four strategies, given up against wasted invocations for consistently-over-ceiling agents (self-limiting) and a deterministic fallback that is not market-informed.
- **Decision:** `_over_ceiling(amount, budget_limit)` helper checked in every write-back; above-ceiling bids priced at the deterministic fallback with typed source labels (e.g. `spot_bidding_ceiling_clamp`).
- **Results:** Shipped PR #143 (merged 2026-07-05) with 6 tests; the agent owns pricing within [0, budget_limit], the orchestrator owns the ceiling.


## 13. Test Tenant & Platform Data

### AD-075 — Source Test-Tenant Data from Three Public Datasets; Extract Golden Sets from the Same Source
**Status:** Accepted · **Theme:** 13. Test Tenant & Platform Data · **PRD:** PRD-012

- **Problem:** A demonstrable dataset covering all four Kraljic quadrants plus golden eval data was needed without real customer procurement data (licensing/privacy) and without synthetic data distorting the Kraljic distribution.
- **Alternatives:** Real customer data (rejected: unavailable in demo/CI); fully synthetic (rejected: can't reproduce Kraljic distribution/performance statistics); no dedicated dataset (rejected: unreproducible CI evals and demos).
- **Trade-offs:** Public, reproducible, license-free source vs. reduced realism, mild train-on-test risk (mitigated by stratified sampling), and load-bearing phase ordering.
- **Decision:** Load Kaggle Kraljic (~1k), Kaggle Procurement KPI (~5k), UCI Online Retail (~541k) in strict phase order via the Test Tenant Skill; extract stratified golden sets (200/500/5,000 records) from the same source.
- **Results:** `skills/test_tenant/` loaders + `extract_golden_datasets`; quality assertions gate each phase; golden files to S3 for Strands Evals (AD-90). This is the legacy direct-to-domain path: the master-data path (AD-76/AD-77) consumes the same CSVs, but no shadow validation has ever run — after AD-77's 2026-08-13 runtime deploy it is reachable yet still unverified, so this loader remains the only path known to have loaded data.

### AD-076 — Make the Master Data Store the Source of Identity Using Deterministic uuid5
**Status:** Accepted · **Theme:** 13. Test Tenant & Platform Data · **PRD:** PRD-015

- **Problem:** The ERP emulation needs stable entity IDs surviving re-ingestion and cross-dataset resolution (same supplier in Phase 1 and Phase 2), so the PR→Negotiation→Award→PO trace chain stays unbroken.
- **Alternatives:** Auto-generated uuid4 (rejected: non-deterministic — duplicate IDs); centralized ID registry service (rejected: unneeded infrastructure); Buyer Team minting IDs (rejected: defeats the canonical master store).
- **Trade-offs:** Stable idempotency keys and cross-dataset resolution with no registry vs. opaque IDs and rename instability (a genuine rename mints a new entity).
- **Decision:** Mint all IDs as `uuid5(NAMESPACE_DNS, f"{tenant_id}:{entity_type}:{name}")`; Buyer Team's `ingest_*` tools preserve master IDs verbatim.
- **Results:** Shared formula across legacy and master loaders; master store is the read source via `tenant-mdm-emulator` (AD-77); stable integration idempotency keys `(tenant_id, source_system, external_id=master_id)` underpin the demo's capstone PR→PO trace.

### AD-077 — Two New MCP Servers and Event-Driven PR Pickup via DynamoDB Streams
**Status:** Accepted (runtime now deployed — see the 2026-08-13 update) · **Theme:** 13. Test Tenant & Platform Data · **PRD:** PRD-015

- **Problem:** The test tenant must emulate a real ERP — external system owning master data and emitting PR events — rather than writing directly into Buyer Team domain tables, or the demo validates a shortcut no production tenant would use.
- **Alternatives:** Direct domain-table writes (rejected: bypasses the real integration path); scheduled polling (rejected: latency, not event-driven); extending the domain MCP server (rejected: conflates master and domain state, bigger blast radius).
- **Trade-offs:** Real ERP-shaped integration path vs. more moving parts (2 servers, a Stream, a router Lambda, a DLQ); the router's re-fetch-current-state invariant makes at-least-once delivery safe at the cost of intentional no-op re-invocations.
- **Decision:** Two new MCP servers — `dynamodb-master-data` (internal CRUD) and `tenant-mdm-emulator` (ERP-shaped read API) — plus a DynamoDB Stream on the master PR table triggering a `pr-event-router` Lambda that re-fetches current state and invokes `ingest_purchase_requisitions`.
- **Results:** Platform MCP total 6→8; router deployed with retries, bisect-on-error, and an alarmed DLQ. Correction 2026-08-01 (PR #250): the event-driven half was real, but `dynamodb-master-data` had never run — no runtime, no gateway target — and in-process execution exposed two latent crash bugs (float→Decimal, missing `ExpressionAttributeValues`), both fixed; the shadow-validation claim was therefore false. Update 2026-08-13 (PRs #281–#283): both servers now have deployed AgentCore Runtimes (plus `step_functions_orchestrator` in the same effort, PR #284); first deploy surfaced and fixed three latent build bugs (dockerignore, mcp 2.0 API break, missing region_name). A shadow-comparison run is still unattempted.

### AD-078 — Synthetic GSI Sort Keys and Composite Cursor Pagination
**Status:** Accepted · **Theme:** 13. Test Tenant & Platform Data · **PRD:** PRD-015

- **Problem:** `tenant-mdm-emulator` must return entities in stable paginated order like a real ERP read API, but DynamoDB offers no ordering beyond the sort key and naive `LastEvaluatedKey` pagination skips/repeats rows under concurrent writes.
- **Alternatives:** Native `LastEvaluatedKey` (rejected: unstable under concurrent modification); offset pagination (rejected: no native offset, full scans at ~541k rows); bare timestamp sort (rejected: same-millisecond ties are non-deterministic).
- **Trade-offs:** Deterministic ordering and skip-free pagination vs. write amplification on every mutation and three schema attributes existing solely for ordered reads — enforced at the MCP server level, not by convention.
- **Decision:** Synthetic GSI sort keys `lm_sk` (`{last_modified}#{entity_id}`), `cat_sk`, `status_sk`, populated atomically on every Put/Update; composite cursor pagination on `(last_modified, entity_id)` with a base64 `page_token`.
- **Results:** Backs the four `list_*` tools in `tenant-mdm-emulator` (AD-77); the composite cursor is the contract for the Test Tenant App's dataset views and for Buyer Team's incremental `ingest_*` sync via `since`.


## 14. Test Strategy & Quality Gates

### AD-088 — Five-Layer Test Pyramid with Distinct Gates per Layer
**Status:** Accepted · **Theme:** 14 Test Strategy & Quality Gates · **PRD:** PRD-008

- **Problem:** Tests vary from milliseconds (unit) to tens of minutes (load); running everything at every stage is unusably slow, running too little lets regressions reach expensive stages — each failure mode needs the cheapest stage that can detect it.
- **Alternatives:** Single staging gate (rejected: no PR feedback, slow pipelines); flat suite everywhere (rejected: loses fast feedback); no action (rejected: inconsistent gating per team).
- **Trade-offs:** Gained fast PR feedback (< 5 min) and fail-fast at the cheapest capable stage; given up later detection of integration (G5) and E2E (G6) bugs, and no chaos coverage in v1.0.
- **Decision:** Five-layer pyramid (Unit → Integration → E2E → Load → Chaos) with distinct gates: Unit blocks PR (G1), Integration blocks dev deploy (G5), E2E blocks staging (G6), Load nightly advisory, Chaos deferred to v1.1; 105 cases in v1.0.
- **Results:** Realized as gates G1–G8 in GitHub Actions; environment split by AD-89, PR-quality gate by AD-90; inventory owned by PRD-008 §9.

### AD-089 — LocalStack for Integration; Real Bedrock (SimpleLLM) for E2E
**Status:** Accepted · **Theme:** 14 Test Strategy & Quality Gates · **PRD:** PRD-008

- **Problem:** Integration tests need realistic AWS behavior but real Bedrock on every dev deploy is too costly/slow; mocking the LLM for E2E would defeat E2E's purpose — the mock strategy must be cost-efficient at integration and model-realistic at E2E.
- **Alternatives:** Real AWS for integration (rejected: too slow/expensive); mock everything (rejected: can't exercise cross-component A2A); full DefaultLLM for E2E (rejected: prohibitively expensive); no action (rejected: either too expensive or insufficient realism).
- **Trade-offs:** Gained realistic integration without inference cost and genuine model validation before staging; given up LocalStack fidelity gaps and less continuous coverage of the strongest model tier.
- **Decision:** LocalStack (DynamoDB/S3/SQS) + real A2A in Docker for integration (gated G5); real AWS with SimpleLLM tier for E2E (gated G6); `BUYER_TEAM_TEST_MODE=mock|live` and an independent `BUYER_TEAM_MEM0_MODE` toggle.
- **Results:** Realized via `docker-compose.test.yml`; Mem0 live-endpoint exercised at E2E only when `mem0_enabled=true` (E2E-01–04, IP-1/2/4). Implements AD-88's gate structure and supports AD-90's golden-dataset path.

### AD-090 — Strands Evals on Every PR
**Status:** Accepted · **Theme:** 14 Test Strategy & Quality Gates · **PRD:** PRD-008

- **Problem:** Agent quality can regress from prompt/tool/config changes unit tests can't see; catching it only at staging means the causal commit is days old — evals must run at the PR.
- **Alternatives:** Staging-only (rejected: late, expensive diagnosis); nightly-only (rejected: regressions accumulate); full DefaultLLM per PR (rejected: exceeds the 2-min budget); no action (rejected: too slow, too late).
- **Trade-offs:** Gained quality as a hard merge requirement caught where the author has context; given up PR latency and a fast-screen fidelity gap versus the full G7 suite (by design — the gates compose).
- **Decision:** Evals against a version-pinned golden dataset on every PR with SimpleLLM, < 2-min budget, ≥ 95% accuracy gate (G3) blocking merge; full suite at staging (G7) blocking promotion.
- **Results:** Never built as described — no Strands EvalSuite or Git-LFS exists. As-built: `run_all.py --ci` offline fixture checks + `judge-smoke` (4 Bedrock calls, PR #193) at the PR; weekly `staging-eval-suite.yml` at G7, made blocking by PR #185 and enforced by `verify-staging-gate` (PR #298, exact-SHA check added PR #304) — intent satisfied by simpler mechanisms. G7 was nonetheless structurally unpassable from PR #298 until impl PR #332 (2026-08-19): every run failed on calibration measured against placeholder human labels, so the gate blocked prod promotion for a fixture reason rather than a quality one (AD-32, AD-143).

