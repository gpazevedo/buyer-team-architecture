# ADR-099: Progressive Disclosure for MCP Skill Runtime

**Status:** Accepted  
**Date:** 2026-06-21  
**PR:** [#41](https://github.com/gpazevedo/buyer-team-impl/pull/41)

## Context

The skill runtime (`dev_skill_runtime`) exposed MCP tools (`ingest_purchase_requisitions`,
`reset`, `load_datasets`, `validate_datasets`) without a discovery mechanism. Clients
(MCP hosts, agents, humans) had no way to enumerate available capabilities or read
usage documentation without consulting external PRD files.

## Decision

Implement a **progressive disclosure** pattern with three levels:

| Level | Tool | Returns | Cost |
|-------|------|---------|------|
| L1 | `catalog()` | `[{name, summary}]` — capability list from in-memory manifest | Zero network calls |
| L2 | `skill_manual(capability)` | `{capability, skill, manual}` — full `SKILL.md` text | Local file read |
| L3 | Existing tools | Tool execution | Full invocation |

- **L1 `catalog()`** reads from a `_CAPABILITY_MANIFEST` dict in the server module
  (no file I/O, no network) and returns every capability's name and one-line summary
- **L2 `skill_manual(capability)`** resolves the owning skill from the manifest,
  reads the co-located `impl/skills/<skill>/SKILL.md` file, and returns its full text
- **L3** is the existing MCP tool surface — unchanged
- `SKILL.md` files are co-located with each skill under `impl/skills/<skill>/` and
  are the authoritative usage spec (parameters, behavior, idempotency, errors)

## Consequences

- **Discoverable** — clients can enumerate capabilities before loading full documentation
- **Cheap L1** — no network or file I/O, safe for every connection handshake
- **Self-documenting** — `SKILL.md` is a single canonical source readable by both
  humans and agents; adding a new capability means writing its SKILL.md and adding
  one entry to the manifest
- **Unknown capability** returns a structured error dict, not a crash
- The manifest must be kept in sync with actual tools — out-of-sync is a documentation
  bug, not a runtime failure
