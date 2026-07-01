# AD-106 — Tool-Output Compaction via AfterToolCallEvent Hook

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-106 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-059, AD-065, AD-101

## Context

Strategy agents (Leverage, Bottleneck, Strategic Partnership, Bid Evaluation) make multiple tool calls within a single Agent invocation. Each tool result — DynamoDB scans, supplier lookups, TCO calculations — is appended to the LLM conversation context for subsequent calls. Without a bound on tool-output size, a single large result (e.g., a 50 KB supplier catalog) can inflate every subsequent LLM call within that invocation, multiplying token cost across the remaining tool-call rounds.

The per-session cost ceiling (`max_session_input_tokens`, AD-048) caps total session spend but does not prevent the steady inflation of per-call input tokens as accumulated tool outputs grow. Prompt caching (AD-059) reduces the cost of the invariant prefix (~90% savings) but does not constrain the variable, growing portion of the context — the tool results themselves.

## Decision

A Strands `AfterToolCallEvent` hook (`compact_tool_output`) truncates tool results in-place when their serialized JSON exceeds `MAX_TOOL_OUTPUT_CHARS` (default 4000). String values longer than 500 characters are truncated **head + tail** within that budget (head 2/3, tail 1/3, joined by a `…[N chars elided]…` marker); the result structure (keys, list lengths) is preserved. Keeping the tail matters because a tool output's answer — running total, final status — often sits at the end of a long field, which head-only truncation would drop. A `_compacted` flag and `_original_size` are set on dict results so downstream code can detect truncation.

The hook is registered in `AgentBlueprint.new_agent()` alongside `log_tool_call` in the shared `buyer_agent_core` package (AD-101), so all 7 agents inherit it uniformly. Controlled via environment variable; set `MAX_TOOL_OUTPUT_CHARS=0` to disable.

## Alternatives Considered

- **Orchestrator-level compaction (trim agent input before the next A2A call).** Rejected: BT agents are invoked once per negotiation, with multi-round tool logic internal to the agent via Strands' tool loop. Compaction must operate at the agent/tool level, not the orchestrator.
- **Per-agent prompt-builder truncation.** Rejected: puts the burden on each agent's `prompt_builder`, creating 7 duplicate implementations and a risk that a new agent forgets it. The shared hook is the single point of enforcement.
- **AgentCore Memory as the compaction target.** Rejected: would shift cost from LLM tokens to AgentCore Memory storage without reducing total cost; adds complexity without material benefit.
- **Summarization-based compaction (LLM call to summarize tool output).** Rejected: replaces token bloat with a summary-model call — trading one cost for another. Truncation is deterministic, zero-token, and preserves structure.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Bounded per-call input-token growth across multi-round tool loops — large tool results truncated before they inflate every subsequent call | Truncated tool results may omit the *middle* detail of a long field; head + tail preserves the start and end (totals/status) but the elided span is still lost to later reasoning rounds |
| Zero-token, deterministic, runs inline on every tool call | The `_compacted` / `_original_size` keys are reserved; a tool result schema must not use them |
| Single point of enforcement in the shared factory — no per-agent duplication | The 500-char string limit and 4000-char output threshold are fixed defaults; per-agent tuning requires env-var or code changes |

The information-loss risk is mitigated by the 4000-char default: a well-structured tool result under 4 KB of JSON typically carries enough signal for agent reasoning. The `_compacted` flag lets future hooks or evaluators detect truncation and flag quality risks.

## Results

Implemented in `packages/buyer_agent_core/buyer_agent_core/compaction.py` — the hook runs after every tool call and mutates `event.result` in place. Registered in `AgentBlueprint.new_agent()` (`factory.py:49`), which is the single assembly point for all 7 agents. The cache-prefix purity invariant (AD-028, AD-065) is unaffected — the hook fires post-tool, not during request assembly.

Covered by 9 tests in `packages/buyer_agent_core/tests/test_compaction.py`: truncation unit tests, hook behavior (below-threshold no-op, above-threshold truncation, disabled, None-result), and a prefix-purity regression test confirming the cache prefix is unchanged.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
