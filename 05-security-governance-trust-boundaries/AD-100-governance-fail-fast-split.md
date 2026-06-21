# AD-100 — Governance Fail-Fast Split — ConfigUnreachable vs GovernanceKeyMissing

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-100 · **Source PRD:** PRD-005/PRD-010 · **Status:** Accepted · **Related:** AD-48, AD-64, AD-65, AD-25, AD-63

## Context

AD-48 mandates fail-fast on system-config unavailability: an agent must never run on guessed governance. In practice, two distinct failure modes were observed in `dev`:

1. **Row gone** — the entire `governance/default` item is absent from `{env}-system-config` (DynamoDB outage, seed-apply failure, accidental delete).
2. **Block dropped** — the row is present and valid, but a required block inside it (e.g., `approval_thresholds`) was dropped from the JSON during a config deploy — the recurring "governance blocks dropped from default" drift.

A single `ConfigUnreachable` exception masked the difference, making ops triage harder than necessary. Worse, a missing block was silently tolerated if the row existed — callers received a dict and just didn't find the key they expected, defaulting implicitly to whatever the code path happened to use.

## Decision

Split the single fail-fast exception into two distinct types in `orchestrator/graph_common.py`:

- **`ConfigUnreachable`** — the entire config row is absent or DynamoDB is unreachable (infrastructure incident).
- **`GovernanceKeyMissing`** — the row is present but a required block inside it was dropped (config-deploy defect).

Callers declare their required blocks via a `require=` parameter on `load_governance_config`:

```python
def load_governance_config(tenant_id=None, profile="default", *, require=None) -> dict
```

Resolution is two-stage (AD-64): `governance/<profile>` is the base; `governance/tenant#<id>` shallow-overlays it. After resolution, `_require_keys` checks every name in `require` is present in the merged dict — a missing block raises `GovernanceKeyMissing`. Both exceptions fail fast per-request → the node DLQs → `REQUIRES_ATTENTION`; the Lambda runtime stays up and handles subsequent requests normally.

The split sits entirely in the **orchestrator** — agents never see either exception. The agent boot path (`buyer_agent_core/model_resolver.py`) is never-raise: `SystemConfig.tiers()` swallows all read failures and returns `{}`, falling through to env→seed (AD-95). The orchestration-side move (from AD-65's narrowed factory scope) keeps the agent path clean while enforcing governance integrity at the only point it matters — the orchestrator node that uses the governance values.

## Alternatives Considered

- **Single exception with a message string discriminator.** Rejected: requires callers to parse error messages, which breaks on format changes; distinct types let ops dashboards route on exception class.
- **Silent default fallback for missing blocks (the pre-PR-42 behavior).** Rejected: a dropped `approval_thresholds` block that silently fell back to `auto_approve=True` could approve spend with no signal. The whole point of governance fail-fast is that no governance decision is made on guessed values.
- **Validate blocks at deploy time (CI/CD guard).** Rejected as insufficient: deploy-time validation is necessary but not sufficient — a guard can catch the drift before it reaches production, but a runtime check is still needed to catch config changes made outside the pipeline or DynamoDB-level mutations the guard doesn't see. Both layers exist; the runtime check is the last line of defense.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Two distinct failure signals: ops can route `GovernanceKeyMissing` to the config-deploy responder and `ConfigUnreachable` to the infrastructure responder | Two exception types to maintain instead of one |
| A dropped required block is guaranteed to be caught — no silent defaulting | Every caller must declare its `require=` list; a new node that forgets gets no enforcement (test coverage required) |
| The agent boot path stays clean (never-raise) while governance integrity is enforced at the orchestrator — the split aligns with AD-65's narrowed factory scope | Two code locations to reason about for config-plane failure (orchestrator vs agent), though they handle disjoint config groups |
| `GovernanceKeyMissing` surfaces what "the row existed but the block was dropped" means — the config-drift equivalent of a missing row, not a less-severe condition | |

## Results

Implemented in `orchestrator/graph_common.py` (`ConfigUnreachable`, `GovernanceKeyMissing`, `_fetch_config`, `_load_config`, `_require_keys`, `load_governance_config(require=…)`). Covered by `orchestrator/tests/test_governance.py` (row-absent → `ConfigUnreachable`, block-dropped → `GovernanceKeyMissing`, present+complete → pass). The `node_approval_gate` adopts `require=["approval_thresholds"]`. Shipped in PR #42 (DynamicAgentFactory + governance hardening, `feat/dynamic-agent-factory`).

The split is described in PRD-010 §3.2 (Governance Config Loading) and cross-referenced in AD-48 (revised fail-fast locality), AD-64 (two-stage resolution with `require=`), and AD-65 (factory scope narrowed to model+cache).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
