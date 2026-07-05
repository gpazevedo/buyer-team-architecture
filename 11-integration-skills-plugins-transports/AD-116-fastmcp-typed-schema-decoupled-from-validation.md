# AD-116 — FastMCP Tool Params Stay `dict`; `WithJsonSchema` Carries the Real Schema

**Theme:** Integration, Skills, Plugins & Transports
**Catalog:** AD-116 · **Source PRD:** PRD-013 · **Status:** Accepted · **Related:** AD-107, AD-109

## Context

The PO-receiving skill tools (`receive_purchase_order`, `acknowledge_purchase_order`, `get_order_history` in `skill_runtime/server.py`) took bare `dict` parameters, so FastMCP inferred an empty/untyped JSON Schema for MCP/Gateway tool discovery — callers had no way to see the real shape of `purchase_order` short of reading the skill's source. The obvious fix — annotate the parameter with the actual Pydantic model (`purchase_order: IncomingPurchaseOrder`) — was tried first (PR #144). It broke the tool's structured-rejection contract: FastMCP's dispatch layer (`FuncMetadata.call_fn_with_arg_validation`) builds an `arg_model` from the function signature and calls `arg_model.model_validate(...)` *before* invoking the function body; a malformed nested field (e.g. missing `supplier.supplier_id`) raised a `pydantic.ValidationError` there, which FastMCP's generic `except Exception` handler re-wraps as a protocol-level `ToolError` — never reaching `receive_purchase_order`'s own `ValidationError → {"status": "REJECTED", "validation_errors": [...]}` handling. This was reproduced directly against the real registered tool via `server.call_tool(...)`. The tests added alongside the typed-schema change only called the underlying module function directly with a `dict`, so the regression shipped invisibly — it only surfaces when a payload is dispatched through FastMCP itself, which is exactly the path every real MCP/Gateway caller uses.

## Decision

Keep the tool's actual parameter type as `dict` (or `dict | None`) so FastMCP's arg-validation only checks "is this a dict" and never rejects on nested shape. Separately, precompute each Pydantic model's `model_json_schema()`, inline its `$defs`/`$ref`s into a self-contained schema (`_inline_defs`), and attach it purely for discovery via `Annotated[dict, WithJsonSchema(schema)]`. The tool body still constructs the real model (`IncomingPurchaseOrder(**purchase_order)`) and catches `ValidationError`, mapping it to the domain's existing structured-rejection shape. FastMCP/Gateway tool discovery now advertises the true nested schema; runtime validation and its error contract stay entirely inside the tool, under application control.

## Alternatives Considered

- **Type the parameter as the Pydantic model directly (`purchase_order: IncomingPurchaseOrder`).** Rejected: proven to raise `ToolError` at FastMCP's dispatch layer before the function body runs, replacing the structured `REJECTED`/`validation_errors` response with an opaque protocol-level failure — verified via a direct `call_tool` reproduction against the real tool.
- **Keep params as bare `dict`, no schema annotation (status quo before this decision).** Rejected: FastMCP then advertises an empty object schema for the tool, so MCP/Gateway clients (including the AgentCore Gateway's own tool-discovery surface) can't see the real shape of `purchase_order`/`ack`/`filters` without reading source — defeats the point of typed schemas (mirrors the OpenAPI request-body pattern this work was adopting from the MRO reference doc).
- **Validate manually inside the tool and skip `WithJsonSchema` entirely, leaving FastMCP's inferred schema empty.** Rejected: same discoverability gap as the status-quo alternative; the whole motivation was a real, inspectable schema at the MCP/Gateway boundary, not just internal validation.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Real, inspectable JSON Schema at the MCP/Gateway tool-discovery boundary, with the graceful `REJECTED`/`INVALID` contract fully preserved on the real dispatch path | Schema and runtime validation are now deliberately decoupled — a maintainer reverting the parameter type to the Pydantic model "to make it stricter" silently reintroduces the `ToolError` regression |
| No change to the tool's external error contract — existing callers see the same response shapes they always did | Requires a hand-rolled `_inline_defs` helper, since `WithJsonSchema` does not carry a model's `$defs` dictionary forward; every new typed tool schema must be pre-inlined at module load |

The decoupling is deliberate but fragile: it depends on nobody "simplifying" the annotation back to the model type. A real-dispatch regression test is the guard against exactly that, not code review alone.

## Results

Applied to all three PO-receiving tools in `skill_runtime/server.py`: `receive_purchase_order` (`IncomingPurchaseOrder`), `acknowledge_purchase_order` (`AckPayload`), `get_order_history` (`OrderHistoryFilters`). Verified with real-dispatch tests in `skill_runtime/tests/test_po_receiving.py` — `test_real_server_dispatch_receive_well_formed_po`, `_acknowledge`, `_get_order_history`, and `test_real_server_dispatch_receive_malformed_po_still_rejected` — all importing `skill_runtime.server` and calling `server.call_tool(...)` directly against the actual registered tools, rather than a parallel construction, so a future regression back to bare model types is caught. Establishes the pattern any future FastMCP tool in this codebase should follow when it needs a real schema without inheriting FastMCP's protocol-level strictness. Complements AD-107 (wires tenant context into FastMCP servers; cited type-hint-driven schemas as an unqualified benefit of FastMCP without addressing validation failure modes) and AD-109 (typed ack/reject, though at the `test_tenant_app` REST layer rather than this MCP skill layer).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
