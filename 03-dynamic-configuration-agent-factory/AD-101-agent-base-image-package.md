# AD-101 — Agent Base Image + Shared Package Delivery via Immutable `agent-base`

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-101 · **Source PRD:** PRD-003/PRD-010 · **Status:** Accepted · **Related:** AD-25, AD-65, AD-95, AD-57, AD-54, AD-104

## Context

Before PR #42, each of the 7 LLM agents had its own vendored copy of `config.py` (resolving model tiers), `observability.py` (OTEL setup), and a Dockerfile that independently `uv pip install`'d the same Python dependencies (Strands, Bedrock, OTEL, boto3). This caused three problems:

1. **Duplication drift** — a fix to the model-ladder (AD-95) or the cache-prefix invariant (AD-65) had to be applied identically across 7 files; one missed copy meant one agent ran different logic.
2. **Unpinned dependencies** — each agent resolved its own dependency versions at build time, so two agents built at different times could ship with different Strands/Bedrock versions.
3. **Build waste** — 7 nearly-identical Docker builds installed the same 400MB+ of dependencies 7 times in CI.

AD-65 narrowed the shared factory to model-ladder + cache-prefix only (governance/flags/thresholds moved to the orchestrator). That narrowed scope made a shared package feasible: the factory no longer pulled in governance, so it didn't create a config-plane dependency for agent boots.

## Decision

Ship a single immutable `agent-base` Docker image and a shared `buyer_agent_core` Python package. All 7 agent images `FROM agent-base:<git-sha>` and COPY only their own thin files.

**Shared package (`impl/packages/buyer_agent_core/`):**
- `factory.py` — `DynamicAgentFactory` + `AgentBlueprint` (model-ladder + cache-prefix only)
- `model_resolver.py` — `SystemConfig` + `resolve_model_id` (boot-safe, never-raise)
- `cache.py` — `cached_system_prompt`, `model_cache_kwargs`
- `seed_mirror.py` — `SEED_TIERS` (in-image mirror of the seed config)
- `spec.py` — `AgentSpec` dataclass (frozen, declarative per-agent definition)
- `serve.py` — shared A2A serving layer (`make_runner`, `build_app`)
- `observability.py` — `setup_telemetry`, `log_usage`, `log_tool_call`

**Base image (`impl/docker/agent-base/Dockerfile`):**
- Multi-stage build: `builder` stage installs dependencies via uv, `runtime` stage copies only installed packages and entry-points (drops uv binary from final image)
- `FROM python:3.14-slim` both stages; non-root `agentuser` in runtime
- Pre-compiles bytecode at build time (`UV_COMPILE_BYTECODE=1`), uses `UV_LINK_MODE=copy` for clean multi-stage transfers
- `RUN --mount=type=cache,target=/root/.cache/uv` for cached package downloads across rebuilds
- Installs `strands-agents[a2a,otel]`, `a2a-sdk[http-server]`, OTEL distro, `boto3`, `uvicorn` in the builder
- Installs `buyer_agent_core` system-wide in the builder (so `import buyer_agent_core` resolves in every per-agent image)
- Sets OTEL env vars (distro, configurator, OTLP endpoint, semconv opt-in)
- `EXPOSE 9000`

**Per-agent image (e.g., `impl/agents/kraljic_classifier_llm/Dockerfile`):**
```dockerfile
ARG AGENT_BASE=agent-base:latest
FROM ${AGENT_BASE}
COPY agent_spec.py agent.py /app/
COPY tools.py system_prompt.py response.py /app/
ENV OTEL_SERVICE_NAME=kraljic-classifier-llm
CMD ["opentelemetry-instrument", "python", "-m", "buyer_agent_core.serve", "agent_spec.py"]
```

**AgentSpec pattern** — each agent declares itself as a frozen dataclass:
```python
from buyer_agent_core.spec import AgentSpec
from buyer_agent_core import Tier

spec = AgentSpec(
    name="kraljic_classifier",
    tier=Tier.SIMPLE,
    system_prompt=SYSTEM_PROMPT,
    tools=[classify_kraljic],
    plugins=no_plugins,
    response_model=KraljicClassification,
)
```

**Bootstrap script (`scripts/bootstrap_ecr_images.sh`):** builds the base image first, then each agent image with `--build-arg AGENT_BASE=agent-base:<git-sha>`, pushing all to ECR.

**Immutability:** the tag is the git SHA (or semver), never `latest`. Agents pin the exact base tag they were tested against. CI builds the base once per push; agent builds consume the already-built base.

## Alternatives Considered

- **Keep per-agent duplication with a CI copy-guard.** Rejected: the CI guard catches drift but doesn't eliminate the 7× maintenance burden or the 7× build waste; a shared package is the simpler solution.
- **Monorepo-wide shared package outside the agent tree.** Rejected: the package needs to be COPY'd into the Docker build context; co-locating it under `impl/packages/` keeps the build self-contained with no external path references.
- **`FROM python:3.13-slim` in every agent (no base image).** Rejected: reverts to the pre-PR-42 duplication; renounces the single canonical copy and the build-time dependency freeze.
- **Use `latest` tag for the base.** Rejected: `latest` is not reproducible — a rebuild of the base changes every downstream agent image without a code change. Immutable SHA tags (AD-54 build-once-promote) ensure the agent image is a deterministic function of its source.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One canonical copy of the agent runtime stack — a fix to any shared module is deployed to all agents by rebuilding only the base | The base image is a new build artifact with its own CI step; a base change triggers agent rebuilds |
| Per-agent Dockerfiles shrank from ~40 lines to ~10; per-agent `config.py`/`observability.py` duplicates eliminated | Agents must conform to the `AgentSpec` contract — implicit `config.py`-style variation is no longer possible |
| Immutable SHA tag guarantees the tested base is the deployed base; no floating `latest` surprise | The base must be built before any agent, serializing a previously parallel build step (mitigated by the bootstrap script and ECR caching) |
| CI can validate that a tagged base is present (placement-guard) and that no agent imports outside `buyer_agent_core` (drift-guard) | Two new CI guard tests to maintain |

## Results

Implemented in PR #42 (`feat/dynamic-agent-factory`): `impl/packages/buyer_agent_core/` (shared package), `impl/docker/agent-base/Dockerfile` (base image), `scripts/bootstrap_ecr_images.sh` (build orchestration). All 7 agent `Dockerfile`s thinned to the pattern above. `impl/agents/kraljic_classifier_llm/agent_spec.py` + `agent.py` serve as the reference thin-agent example. Placement-guard test verifies no agent imports from outside `buyer_agent_core`; drift-guard CI validates that the base tag is the current SHA.

Base image optimized in PR #75 (`perf/improve-dockerfiles`): multi-stage build, bytecode pre-compilation, non-root user, and BuildKit cache mounts (see AD-104 for rationale). `skill_runtime/Dockerfile` converted to the same multi-stage pattern.

Cross-referenced in AD-25 (factory scope narrowed, delivered via `agent-base`), AD-65 (model-ladder + cache-prefix only, shared via this mechanism), AD-95 (the boot-safe model ladder in `model_resolver.py` is the single init path, carried in the base image), and AD-104 (Docker image optimization strategy).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
