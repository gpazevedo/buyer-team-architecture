# AD-125 — `response_builder` Replaces the Deprecated `structured_output()` Second Bedrock Call

**Theme:** Agent Architecture & Behavioral Control
**Catalog:** AD-125 · **Source PRD:** PRD-003 · **Status:** Accepted · **Related:** AD-22, AD-13, AD-57

## Context

Every agent's run had two phases: an analysis phase (`agent(prompt)`, the tool-calling loop grounded in live data, AD-22) followed by a coercion phase (`agent.structured_output(response_model, ...)`) that re-sent the entire completed transcript back to Bedrock purely to force it into the agent's Pydantic response schema. The second call adds no reasoning — by the time it runs, every fact the response needs already exists as a tool-call argument or result in the transcript from phase one. It doubles Bedrock cost and latency per agent invocation for no accuracy gain, and doubles the surface for a hallucinated field (the model can restate a number differently on the second pass than it computed on the first).

## Decision

Add an optional `AgentSpec.response_builder: Callable[[...], ResponseModel] | None`. When set, `serve.py` skips `structured_output()` entirely and instead assembles the Pydantic response in code from the completed run's tool-call transcript, read via `buyer_agent_core.tool_transcript.last_tool_call`/`all_tool_calls`. For agents where every response field already flows through an existing domain tool's arguments or results (`bottleneck_negotiation_llm`, `strategic_partnership_llm`, `spot_bidding_llm`, `award_comms_llm`), the builder reads that tool's transcript directly. For agents where no domain tool naturally carries the full response — a classification judgment (`kraljic_classifier_llm`) or a compiled cross-round aggregate (`leverage_auction_llm`) — add a thin pass-through tool whose only purpose is forcing the model's judgment through a typed call argument (`submit_kraljic_classification`; `close_auction`'s signature extended to accept the compiled aggregate), and the builder reads that tool call's *input* instead of a result.

## Alternatives Considered

- **Keep `structured_output()` on every agent.** Rejected: it's a second full Bedrock round-trip that re-derives an answer already present in the transcript — pure added cost and latency with no reasoning benefit, and a second chance for the model to restate a fact differently than it computed it.
- **Make every tool call return the full final response inline.** Rejected: couples domain tool contracts (e.g. `send_round_feedback`) to the top-level response schema, which changes independently of what each tool needs to do its own job (AD-22's tools-as-boundaries principle keeps tool contracts scoped to the action, not the caller's eventual output shape).
- **Parse the model's final free-text message with a regex/LLM extractor.** Rejected: reintroduces exactly the prompt-injection and parsing fragility that AD-22's typed-tool boundary was designed to eliminate — a free-text final message is not interceptable by Cedar or a steering hook the way a typed tool call is.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One Bedrock call per agent invocation instead of two — roughly halves per-invocation Bedrock cost and latency for every migrated agent | Two of the six migrations (`kraljic_classifier_llm`, `leverage_auction_llm`) required a new pass-through tool and a prompt change instructing the model to call it as its final action — real model-facing behavior risk, not a pure refactor, validated by live smoke tests rather than unit tests alone |
| Response assembly is testable Python given a transcript, with no re-invocation of the model needed to verify it | `response_builder` trusts the tool-call arguments exactly as the model produced them — a hallucinated argument now flows straight into the response with no second model pass to self-correct it, a check `structured_output()` incidentally provided |

`spot_bidding_llm`'s migration (PR #207) needed `tool_transcript.all_tool_calls` rather than `last_tool_call` — several of its tools fire once per supplier or per extension-request round, so the response has to be assembled from *every* invocation, not just the most recent, tolerating a call whose result carries no JSON content (an "unreachable supplier" failure branch) by returning `None` for that call rather than raising.

## Results

Migrated across three PRs: `bottleneck_negotiation_llm`/`strategic_partnership_llm` (PR #206), `spot_bidding_llm` (PR #207, added `all_tool_calls` to `tool_transcript` and moved `budget_flag`/`budget_excess` computation into `compile_bid_results` itself so it's deterministic rather than a `structured_output`-coercion judgment call), and `award_comms_llm` / `kraljic_classifier_llm` (new `submit_kraljic_classification` tool) / `leverage_auction_llm` (extended `close_auction`'s signature to accept the compiled aggregate) in PR #210 (merged 2026-07-13). All 6 live agents now set `response_builder`; `agent.structured_output()` remains in `serve.py` only as unused fallback code for a spec that doesn't set it — the deprecated path being phased out, not yet deleted. `award_comms_llm`'s runtime is not invoked on the live path at all (Node 7 renders comms from inline templates — see AD-57's Results note); its migration keeps it consistent with the other 5 agents' pattern but has no live cost or latency effect today.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
