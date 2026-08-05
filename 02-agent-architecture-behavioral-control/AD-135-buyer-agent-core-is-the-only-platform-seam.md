# AD-135 — `buyer_agent_core` Is the Agent Layer's Only Seam onto Strands, A2A and boto3

**Theme:** Agent Architecture & Behavioral Control
**Catalog:** AD-135 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-101, AD-21, AD-22, AD-23, AD-25

## Context

AD-101 shipped a shared package and a shared base image to end per-agent duplication of the runtime stack, and its Results claimed a test enforced the boundary: "no agent imports from outside `buyer_agent_core`." No such test ever existed. In the roughly six weeks between AD-101 landing (PR #42, 2026-06-21) and this decision, the duplication AD-101 exists to prevent reappeared one layer up — in exactly the thin per-agent files the base image does not own:

- All six `tools.py` built their own `boto3.resource("dynamodb")` and botocore `Config` — six copies of one retry policy, so a connect/read-timeout change had to be made six times and could drift silently between agents. Each also read `AWS_REGION` at import, freezing the region by module import order.
- All six imported `tool` directly from `strands`.
- Five `steering.py` files imported `Guide`/`Proceed` from `strands.vended_plugins` while subclassing `GuardHandler`, which is ours — so the steering vocabulary and the base class it belongs to arrived from two different places.

The design intent was never ambiguous: an agent module declares *what the agent does* — spec, prompt, schemas, tool bodies, guards — and the platform layer owns *what it runs on*. What was missing is that "the agent layer does not import the platform" is a property no amount of shared-package availability can produce. A convention that is never checked is indistinguishable from no convention.

## Decision

Make `buyer_agent_core` the single seam, and enforce it structurally.

**One construction site.** A new `buyer_agent_core/aws.py` owns the only `boto3` construction in the agent layer: the shared `Config` (10s connect, 30s read, 3 adaptive retries), a cached resource, and a per-name cached `ddb_table(name)`. Region is read on first use rather than at import, so it is no longer a function of import order. `Attr` and `ClientError` are re-exported alongside it, because two agents build scan filters with `Attr` and award-comms branches on `ClientError` to detect a lost conditional-write race — re-exporting them keeps the invariant intact without pretending the query grammar and error surface are not DynamoDB's.

**One import surface.** `buyer_agent_core.__init__` re-exports `tool`, `Guide`/`Proceed`/`Interrupt`, `ddb_table`, `Attr`, `ClientError` and `emit_metric`, so the steering vocabulary arrives from the same place as `GuardHandler`. Agent modules alias on import (`ddb_table as _table`), preserving the module-local seam that roughly 70 existing tests already patch.

**Three structural checks** (`agents/tests/test_agent_layer_boundary.py`), each closing a different escape route:

1. **An allow-list, not a deny-list.** Agent code may import the stdlib, its own flat siblings (read off the directory, since agents import each other flatly with the agent dir as working directory), and exactly three third-party roots: `buyer_agent_core`, `negotiation_schemas`, `pydantic`. A deny-list of known-bad packages lets any *new* platform dependency through silently; an allow-list red-builds until someone adds it deliberately, and that break is the review signal.
2. **No dynamic-import machinery.** `importlib` is carved out of the stdlib allowance, and `__import__`/`eval`/`exec` calls are rejected. None are used today. Asserting they stay unused is what makes check 1 *sound* rather than best-effort: if an `import` statement is the only way to bind a module, then inspecting every `import` statement sees everything.
3. **`__all__` membership.** Checks 1 and 2 both wave through `from buyer_agent_core import boto3` — the root is approved, so the name is never examined. Requiring every imported name to be a declared export separates a sanctioned re-export (`tool`, `Guide`, `Attr`) from a module-level import artifact that merely happens to be reachable. This check is not theoretical: `from buyer_agent_core.aws import boto3` resolves at runtime and passed every earlier draft of the guard.

A fourth test asserts the source glob matched something, so the parametrized checks cannot pass vacuously, and a fifth asserts every name in `__all__` resolves, since `__all__` is the contract check 3 enforces.

## Alternatives Considered

- **Leave it as convention, documented in `CREATING_A_NEW_AGENT.md`.** Rejected: this is the status quo that produced six copies of one botocore `Config` in under a month, while AD-101 stated the guarantee as fact. A documented convention is what failed.
- **Deny-list the known platform packages (`boto3`, `strands`, `botocore`, …).** Rejected: it enforces the boundary only against dependencies someone already thought of. The first genuinely new one — an SDK, a cache client, an HTTP library — passes silently, which is the failure mode that produced this ADR.
- **Lint rule (`ruff` `flake8-tidy-imports` banned-api) instead of a test.** Rejected on the same ground as the deny-list: it configures a set of banned modules, not a set of permitted ones, and it cannot express check 3 (`__all__` membership) at all.
- **Enforce at the packaging boundary — separate installable distributions per layer.** Rejected: the agent images already `COPY` flat files into `/app` and import each other as siblings (AD-101). Real distribution boundaries would mean restructuring the container layout to buy an import guarantee an AST walk gives for free.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One place to tune connection behavior for the whole fleet — a timeout change is one edit, not six that can drift | An indirection: reading a tool body no longer shows you which `boto3` config it runs under |
| A new platform dependency in agent code cannot land silently; it fails CI until someone adds it to the allow-list on purpose | Legitimate new dependencies pay a friction toll — the allow-list is a file someone must edit, and a reviewer must judge |
| The seam's public contract is explicit (`__all__`), so a re-export is a decision rather than an accident of module layout | `__all__` becomes load-bearing: forgetting to add a name there breaks agent imports in a way whose error message points at the guard, not the omission |
| Region resolution is no longer frozen by import order | |

**What this does not prove.** The guard establishes exactly one property: nothing under `agents/` imports outside the seam. It does *not* establish that the agents are portable off Strands or off Bedrock — `response_builder` still reads `agent.messages` in Bedrock Converse shape, and no import analysis can see a data-shape coupling. Claiming portability from this test would repeat AD-101's error in a new form; the claim here is narrower and exact.

## Results

Shipped in impl PR #258 (merged 2026-08-05): `packages/buyer_agent_core/buyer_agent_core/aws.py`, expanded `__init__` re-exports, six `tools.py` and five `steering.py` converted to seam-only imports, `agents/tests/test_agent_layer_boundary.py`, and `packages/buyer_agent_core/tests/test_aws.py` — a moto-backed suite over `ddb_table`, which is the one construction path no agent test can reach because they all patch `_table` with in-memory fakes. That suite takes `aws.py` to 100% and pins the lazy region lookup: reverting it to an import-time constant fails the test.

No behavior change beyond the region-lookup timing. The agent suite passed unmodified — a direct consequence of keeping `_table` as each module's local name — and 1429 tests pass with pyright and ruff clean across `agents/` and `packages/`.

One real type error surfaced while clearing the package's pyright warnings: `GuardHandler.steer_before_tool` declared `tool_use` as `dict` where the Strands base declares the `ToolUse` TypedDict, which is not an assignable override. Correcting it exposed four test fakes passing partial `ToolUse` payloads (missing the required `toolUseId` and `input`), now routed through one helper. The remaining warnings are SDK annotation gaps documented inline — Strands' `@hook` rewraps the method as `_WrappedHookCallable` declaring `-> None`, erasing a coroutine type that `inspect.iscoroutinefunction` confirms is there.

This closes the gap recorded in AD-101's 2026-08-05 correction. It also strengthens AD-22 and AD-23: tools-as-boundaries and steering-over-prompting both assume agent modules are declarative surfaces, and that assumption is now checked rather than asserted.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
