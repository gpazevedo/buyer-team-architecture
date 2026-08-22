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

## Update 2026-08-20 — L2 was broken in every deployed image since creation (impl PR #340)

`_SKILLS_DIR = Path(__file__).resolve().parent.parent / "skills"` computed correctly
against the repo layout (`impl/skill_runtime/server.py` → `impl/skills/`), which is what
every local test ran against. The Dockerfile does `COPY skill_runtime/server.py ./`, so
in the deployed container `__file__` resolves to `/app/server.py` and `.parent.parent`
lands on `/skills` — a path that has never existed in any built image, while the real
skills land at `/app/skills`. `skill_manual(capability)` therefore raised or returned
empty in production from day one; only `catalog()` (L1, in-memory manifest) and L3 (the
tools themselves) worked. Caught during the 2026-08-20 integration review, verified by
building a probe image mirroring the Dockerfile's actual `COPY` layout — not by reasoning
about the path abstractly. Fixed with `_skills_dir()`, which probes `here/skills` then
`here.parent/skills` and returns whichever exists, covering both the repo layout tests
run against and the container layout the image actually ships.
