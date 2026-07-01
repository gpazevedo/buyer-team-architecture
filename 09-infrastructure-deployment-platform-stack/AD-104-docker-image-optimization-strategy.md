# AD-104 — Docker Image Optimization Strategy: Multi-Stage, Bytecode, Non-Root

**Theme:** Infrastructure, Deployment & Platform Stack
**Catalog:** AD-104 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-101, AD-103

## Context

Before PR #75, all service Dockerfiles (`agent-base`, `skill_runtime`, `test_tenant_app/backend`) were single-stage builds that:

1. **Included the uv build tool in the final image** — `COPY --from=ghcr.io/astral-sh/uv:0.9.3` added a 60MB static binary that was never used at runtime.
2. **Re-downloaded packages on every rebuild** — `uv pip install --system --no-cache` discarded the cache, so CI and local builds fetched every package from PyPI every time.
3. **Left bytecode uncompiled** — Python compiled `.pyc` files on first import at container start, adding latency to cold starts.
4. **Ran as root** — the container `USER` was never set, so the process ran with full privileges inside the container.
5. **Sent the full repo as build context** — no `.dockerignore` existed, so 700MB+ of `.venv`, `.git`, `test_tenant_app`, and unrelated directories were sent to the Docker daemon before every build.

These issues mattered because image builds run frequently (every push to CI, plus local development) and cold starts occur on every scale-out event in AgentCore.

## Decision

Apply five optimizations consistently across all service Dockerfiles:

### 1. Multi-stage builds (builder → runtime)

A `builder` stage installs all dependencies using uv, then a clean `runtime` stage copies only the installed packages (`site-packages`) and entry-point binaries (`/usr/local/bin`). The uv binary — and any build-time artifacts — never enter the final image.

**Applied to:** `agent-base`, `skill_runtime`.

`UV_LINK_MODE=copy` ensures packages are physically copied (not hardlinked) so the `COPY --from=builder` works correctly.

### 2. BuildKit cache mounts

`RUN --mount=type=cache,target=/root/.cache/uv` replaces `--no-cache`. The cache persists across builds and is stored by BuildKit, not in the image layer. Second and subsequent builds resolve packages without downloading.

**Applied to:** all `uv pip install` and `uv sync` RUN lines across agent-base, skill_runtime, and test_tenant_app/backend. The frontend gets `--mount=type=cache,target=/root/.npm` for the npm global install and `--mount=type=cache,target=/root/.local/share/pnpm/store` for pnpm.

### 3. Build-time bytecode compilation

`UV_COMPILE_BYTECODE=1` in the builder stage pre-compiles all `.pyc` files at install time. Cold-start imports skip the parse-and-compile phase entirely.

**Applied to:** `agent-base`, `skill_runtime`. `test_tenant_app/backend` already benefits from `uv sync`'s built-in compilation in CI.

### 4. Non-root user

The runtime stage creates a system user via `useradd -r` (no home directory, no login shell) and runs the process as that user via `USER`. Per-agent images inherit the non-root user from the base.

**Applied to:** `agent-base` (`agentuser`), `skill_runtime` (`skilluser`).

### 5. .dockerignore

A root `.dockerignore` excludes everything not explicitly needed by a Dockerfile that uses the repo root as build context. The pattern is whitelist-oriented: deny all of `packages/*` except `packages/buyer_agent_core/`. Also excludes `.venv/`, `.git/`, `test_tenant_app/`, `node_modules/`, documentation, and non-Docker project directories.

### 6. uv version upgrade

`ghcr.io/astral-sh/uv:0.9.3` → `0.11.25` across all Dockerfiles. Newer uv has faster resolution, better caching, and `UV_COMPILE_BYTECODE` support.

### Frontend-specific

- `pnpm` pinned to `11.6.0` (matches `package.json`'s `packageManager` field) with npm cache mount
- `COPY . .` replaced with targeted copies of individual files (`tsconfig.json`, `vite.config.ts`, `components.json`, `index.html`) and `src/` — source changes no longer invalidate the `pnpm install` layer

## Alternatives Considered

- **Keep single-stage, just add a non-root USER at the end.** Rejected: the uv binary still bloats the image by 60MB and the layer architecture doesn't separate build concerns from runtime concerns.
- **Use a distroless base image.** Rejected: Python is not yet available in a maintained distroless image with 3.14; `python:3.14-slim` is the slimmest official image with the required interpreter.
- **Add a `docker-bake.hcl` for parallel builds.** Deferred: the current CI matrix already builds agents in parallel; bake would simplify configuration but is not a performance bottleneck.
- **Share a single cache mount across all builds.** Rejected: BuildKit cache mounts are scoped to a single Dockerfile build; they don't share across different build invocations without a registry cache (which CI already uses via `--cache-from`/`--cache-to`).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Multi-stage removes ~60MB (uv) from each service image; runtime images carry only what the process needs | Two `FROM` stages per Dockerfile — slightly more verbose, more layers in the build cache |
| Cache mounts eliminate re-downloads for local and CI rebuilds; second build of agent-base completes in seconds | Cache mounts only help when the dependency list hasn't changed; a `pyproject.toml` change still triggers a full re-resolve |
| Bytecode pre-compilation shifts latency from cold-start (every scale-out) to build-time (once per push) | Image size increases by ~39MB (`.pyc` files); net savings are ~21MB after uv removal |
| Non-root user reduces the blast radius of a container escape vulnerability | Build-time operations that need root (e.g., installing system packages) must happen before the `USER` directive |
| `.dockerignore` cuts context from ~700MB to ~289KB — builds start almost instantly | Any new Dockerfile that needs a previously-excluded directory must update the `.dockerignore` |

The bytecode-vs-size trade deserves emphasis: the net image is still ~21MB smaller than the single-stage equivalent (60MB uv removed, 39MB `.pyc` added), so we get faster cold starts _and_ a smaller image.

## Results

Implemented in PR #75 (`perf/improve-dockerfiles`, 2026-06-28). Files changed:

- `impl/.dockerignore` (new)
- `impl/docker/agent-base/Dockerfile` (multi-stage, non-root, bytecode, cache mounts)
- `impl/skill_runtime/Dockerfile` (same pattern)
- `impl/test_tenant_app/backend/Dockerfile` (cache mounts, uv upgrade)
- `impl/test_tenant_app/frontend/Dockerfile` (pnpm cache, pinned version, targeted COPY)

The parent ADR (AD-101) describes the _what_ and _why_ of the agent base image; this ADR describes the _how_ — the specific build-stage techniques that make the images smaller, faster, and more secure.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
