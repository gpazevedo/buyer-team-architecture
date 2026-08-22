# AD-151 — Per-Invocation Config Cache, Shared by Both Readers, Carried Along the Inline Chain

**Theme:** Dynamic Configuration & Agent Factory
**Catalog:** AD-151 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-63, AD-48, AD-64, AD-29, AD-31

## Context

Every node reads its behaviour out of `{env}-system-config` (AD-63) rather than out of environment variables, and every read is a live DynamoDB call by design — REQ-R405 requires that a config change take effect without a redeploy, so nothing may hold a value across invocations. What that requirement does *not* say is that the same item must be fetched repeatedly **inside** one invocation, and it was: a single node invocation independently re-read `governance/default` from five call sites (`retry.py`, `agent_invocation.py`, `circuit_breaker.py`, `timeouts.py` via `resilience/config.py`, plus `graph_common`'s own `_fetch_config` path behind `load_governance_config`/`load_registry_config`/tenant overrides), all fetching the byte-identical item. A traced negotiation showed **~28 of 69 `GetItem` calls (~40%)** were this exact duplicate.

Two forces made that worse than a wasted read. First, each `GetItem` is also an X-Ray subsegment, and trace payload turned out to be a hard budget rather than a soft cost — a full-chain trace measured 370.5 KB against a 500 KB cap that had already cost Node 7 every span it emitted (AD-31, AD-29). Second, the config is *invariant across the whole inline chain*: Nodes 1→2→3→5 run seconds apart inside one synchronous Step Functions execution, so four nodes were independently re-fetching the same three blobs (governance/default 2149 B, registry/default 2011 B, features/default 155 B) within the same few seconds.

Two structural details shaped what could be done about it. The two readers fronting the one table have deliberately different transport policies — `resilience.config` runs 5s/no-retries to fail fast per PRD-006 §2.5, `graph_common` runs 10s/30s/3 adaptive — and they have different miss semantics (`_fetch_config` returns `None` when an item is absent; `read_system_config` raises). And `comms_delivery` reaches `read_system_config` through `resilience.circuit_breaker` without importing `graph_common` at all, so neither reader can own the other's cache.

## Decision

Memoize config reads in one store shared by both readers, scoped to a single Lambda invocation, and carry that store forward along the inline node chain.

- A new `orchestrator/config_cache.py` holds the cache and nothing else. It sits *below* both readers and imports neither, so `comms_delivery`'s import graph is unchanged. Each reader still passes its own DynamoDB Table in, keeping its own timeout/retry policy and its own miss semantics.
- The cache is keyed `(config_group, config_key)` and loads **a whole group per DynamoDB `Query`**, not a `GetItem` per key: a group is a handful of items (a `default`/profile base plus any `tenant#<id>` overlays), and a caller wanting the base almost always wants the overlay too — so one Query replaces the 2–3 GetItems it stands in for, and an **absent overlay costs no read at all**.
- `graph_common.handle_config_errors` — the one decorator already wrapping all six `node_*.py` `lambda_handler`s — calls `config_cache.clear()` at the top of every invocation. **That call is the REQ-R405 freshness boundary**, stated as such in the module docstring.
- Node 1 dumps the cache into `result["_config_snapshot"]` (flat `{group: {key: value}}`) and Nodes 2/3/5 prime from it before their first config read, via `trace_helpers.carry_config_snapshot`/`prime_config_snapshot` — siblings to the existing `inject_trace_context`/`carry_correlation_id`. A `(group, key)` the snapshot does not cover still falls through to a live read.
- **Node 6 (ApprovalGate) deliberately never primes.** Its approval-threshold check must read current config, because its decision may follow an arbitrarily long human-review wait and a Node-1 snapshot carried across that wait would delay a mid-review config change taking effect. Node 7 never calls these helpers either way.

## Alternatives Considered

- **Cache with a TTL across invocations (warm-container memoization).** Rejected: directly contradicts REQ-R405 — a warm container would serve its first invocation's values for the container's lifetime, which is exactly the staleness this platform's config-as-data model exists to avoid.
- **Put the cache inside one reader and have the other call through it.** Rejected: `resilience.config` would then have to import `graph_common` (or vice versa), pulling the whole orchestrator graph into `comms_delivery`'s package, and it would force one transport policy onto both readers — handing `graph_common` the no-retry policy turns a transient throttle into the unhandled SFN `Failed` dead-end that `handle_config_errors` exists to prevent.
- **Read config once in Node 1 and pass only the resolved values forward, dropping the per-node reads entirely.** Rejected: it makes every downstream node structurally unable to read config, which breaks Node 6's requirement to re-read after a human-review pause and leaves no fall-through for a key the snapshot never covered.
- **Status quo.** Rejected: ~40% of a negotiation's DynamoDB calls, and the X-Ray subsegments for all of them, bought nothing.

## Trade-offs

| Gained | Given up |
| --- | --- |
| **5 `GetItem`s per node → 3 `Query`s**, live-measured against `dev-system-config`; the duplicate reads and their trace subsegments are gone | A second freshness boundary to reason about: correctness now depends on `clear()` running before every handler, where before it depended on nothing |
| Trace payload falls to 169.9 KB (from 183.9 KB after the App Signals cut alone) — the last −4% of a −54% reduction against a hard 500 KB cap | The inline chain now carries config *data* in its Step Functions payload, not just identifiers — small (~4.3 KB against a 256 KB per-state I/O cap) but a new coupling between config size and state size |
| Both readers keep their own transport policy and miss semantics; `comms_delivery`'s import graph is untouched | The cache/transport split is a non-obvious layering that a future reader may "simplify" back into one of the readers |
| A group's absent tenant overlay now costs zero reads instead of a wasted `GetItem` | Group-granular loading means an unrelated key's presence in the same group is fetched whether or not this invocation wants it |

Two consequences worth stating plainly. The freshness guarantee is **unchanged, not weakened**: REQ-R405 is about not carrying a value *across* invocations, never about re-fetching an unchanged item within one — and Node 6, the only node whose invocation can straddle a long wall-clock gap, is excluded from priming for exactly that reason. And the snapshot is version-fragile by construction: an execution in flight across a deploy carries whatever shape the previous code wrote, so priming a stale shape yields entries under group names nothing looks up and the node simply reads live. Inert, and only for the length of the deploy window — but it means the snapshot may never become a required input.

## Results

Shipped across impl PR #350 (per-invocation memoization in both readers, plus an autouse `conftest.py` fixture clearing both caches between tests after a leaked-cache failure surfaced the same isolation need the production decorator provides), PR #351 (`_config_snapshot` carried through Nodes 1→2→3→5, `trace_helpers.prime_config_snapshot`/`carry_config_snapshot`), and PR #352 (`orchestrator/config_cache.py` as the single shared store, Query-by-group replacing GetItem-by-key, flat snapshot shape). Live-verified against `dev-system-config`: 3 Queries per node, shared cache confirmed, 633 tests green.

One IAM gap this would otherwise have hit in production surfaced with the Query change: `comms_delivery`'s role was scoped to `dynamodb:GetItem` on the system-config table and reaches `read_system_config` via `resilience.circuit_breaker`, so it needed `dynamodb:Query` added; the node Lambdas already had it via `table/*`, and approval-sweep never reads config. **A read-shape change is an IAM change** — worth remembering wherever a narrowly scoped role fronts this table.

The trace-payload half of the motivation is recorded on [AD-029](../07-observability-evaluation/AD-029-four-layer-observability.md) (Application Signals off, the three-variable set) and [AD-031](../07-observability-evaluation/AD-031-w3c-traceparent-propagation.md) (the 500 KB cap that made payload a budget in the first place).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
