# ADR-0001: Apex Agent Backend for NanoClaw

**Status**: Accepted
**Date**: 2026-05-21
**Slug**: `apex-agent-backend`
**Author**: solutions@getapexinsights.com
**Reviewer**: solutions@getapexinsights.com
**Implementor**: solutions@getapexinsights.com
**Companion Plan**: [`.claude/plans/apex-agent-backend-plan.md`](../plans/apex-agent-backend-plan.md)

> Status transitions: `Proposed` → `Accepted` → `Implemented` → (optional) `Superseded by ADR-MMMM`.
> `promote-to-loop.sh` refuses to run until status is `Accepted`.

---

## TL;DR

- Add Apex as a swappable per-agent-group backend that runs as its own Docker image alongside `nanoclaw-agent`. Switch via a new `container_configs.backend` column flipped through `ncl`.
- Apex speaks NanoClaw's existing two-DB protocol (`inbound.db` / `outbound.db`). Zero new transport, zero host-side rewrite — `src/container-runner.ts` only forks which image and which command it spawns.
- MCP-native tool dispatch and Anthropic prompt caching are first-class in v0.1, not deferred. Telemetry defaults to local SQLite + stdout (no GCP required); Firestore/BigQuery/OTel are optional adapters.
- Optional `APEX_NANOCLAW_AWARE=true` env flag composes an Apex SOUL on the fly from a mounted `groups/<folder>/CLAUDE.md` and `skills/` — letting an existing NanoClaw agent group flip to Apex without re-authoring config.
- Default install behavior unchanged: backend defaults to `nanoclaw`, apex image only built when `INSTALL_APEX=true`. Existing NanoClaw users see no difference until they opt in.

---

## Context (SPARC: Specification)

### Why this matters

NanoClaw v2 ships with one agent provider (Claude Agent SDK via Bun). Adding alternative LLM behavior today means modifying the in-container `agent-runner` directly — disruptive, hard to A/B, and forces every future LLM feature (caching, gates, judge, drift, multi-provider) to be reimplemented per backend.

Apex Insights wants safety gates, telemetry, prompt caching, judge sampling, drift detection, and budget enforcement around every LLM call. Bolting these onto each agent-runner branch (Claude SDK today, OpenCode next, future providers later) duplicates work and fragments observability.

Cleanest decoupling: keep NanoClaw as the channel/session/container host (it is already good at that), and let a parallel agent backend own the LLM pipeline. NanoClaw already supports per-group container customization via `container_configs` (migration 014); this decision extends that surface with one column and one fork in `container-runner.ts`.

### Requirements

| ID | Priority | Requirement |
|----|----------|-------------|
| R1 | MUST | `apex-agent` Docker image polls `inbound.db`, calls Claude via apex.llm pipeline, writes `outbound.db` with valid odd-parity seq numbers. Indistinguishable to NanoClaw's host code from `nanoclaw-agent`. |
| R2 | MUST | Per-agent-group backend selection via `container_configs.backend` column (`nanoclaw` or `apex`). Flippable through `ncl groups config update --backend`. |
| R3 | MUST | Anthropic prompt cache breakpoints inserted by provider adapter based on SOUL config. Telemetry records `cache_creation_input_tokens` and `cache_read_input_tokens` separately. |
| R4 | MUST | Tools defined via `@tool` decorator are auto-registered as MCP servers. Agent loop dispatches all tool calls via MCP — local and remote tools indistinguishable to the loop. |
| R5 | MUST | Telemetry has zero required cloud dependencies. SQLite + stdout sinks ship enabled by default; GCP sinks (Firestore, BigQuery, OTel) activate only when credentials present. |
| R6 | SHOULD | When `APEX_NANOCLAW_AWARE=true`, apex container composes a SOUL from `/workspace/CLAUDE.md` and `/workspace/skills/`. Allows in-place A/B comparison without re-authoring agent config. |
| R7 | SHOULD | `INSTALL_APEX=true` opt-in build flag (default off). Apex image only built when explicitly requested by the operator. |
| R8 | SHOULD | `ncl groups config get` shows both `### NanoClaw composed CLAUDE.md` and `### Apex SOUL config` sections, labeled, regardless of which backend is active. |
| R9 | MAY | Ollama provider in Phase 5 for local-model demo on laptop (validates the portable-framework thesis). |

### Constraints

- Must not modify NanoClaw's two-DB schema (`messages_in`, `messages_out`, `session_state`) or seq-parity convention — NanoClaw host code in `src/delivery.ts` and `src/host-sweep.ts` depends on both.
- Must not introduce a new host-side IPC surface. Apex talks to NanoClaw exclusively through the existing session DB contract — no REST, no gRPC, no shared sockets.
- Must not require Python on the host. Apex builds and runs only inside its own container; the NanoClaw host stays pnpm/Node-only.
- Must reuse NanoClaw's OneCLI gateway for secrets — no direct env-var credential passing into apex containers.
- Must honor `journal_mode=DELETE` on session DBs (NanoClaw's cross-mount visibility invariant — see `container/agent-runner/src/db/connection.ts`).
- Default install behavior unchanged for existing users: backend defaults to `nanoclaw`, apex image not built unless `INSTALL_APEX=true` is set.

### Success criteria

- Flip a test agent group to `backend=apex` and send a message via its wired channel; response is delivered with the same end-to-end semantics as the NanoClaw backend. Runnable: `pnpm test -- src/tests/integration/apex-backend-swap.test.ts`
- Flip the same agent group back to `backend=nanoclaw`; behavior reverts cleanly with no data loss. Runnable: same test, second scenario.
- All host tests pass with apex backend enabled. Runnable: `pnpm test`
- All apex container tests pass. Runnable: `cd container/apex-agent && python -m pytest tests/`
- NanoClaw-aware bootstrap: flipping an existing fully-configured NanoClaw agent group to apex with `APEX_NANOCLAW_AWARE=true` produces a SOUL whose responses are behaviorally consistent with its CLAUDE.md/skills config. Runnable: `pnpm test -- src/tests/integration/apex-nanoclaw-bootstrap.test.ts`
- Telemetry SQLite at `/workspace/.apex/telemetry.db` records one row per LLM call including separate `cache_creation_input_tokens` and `cache_read_input_tokens` fields. Runnable: `pnpm exec tsx scripts/q.ts /tmp/apex-test/.apex/telemetry.db "SELECT cache_read_input_tokens FROM llm_events LIMIT 1"`

---

## Decision

### Pseudocode (SPARC)

```
ALGORITHM apex_runtime_main:
  INPUT:
    - SESSION_DIR (env, default /workspace, holds inbound.db + outbound.db)
    - AGENT_GROUP_ID (env, passed by container-runner.ts)
    - APEX_SOUL_ID (env, default "assistant-base")
    - APEX_NANOCLAW_AWARE (env, default false)
  PRECONDITIONS:
    - inbound.db and outbound.db exist with NanoClaw schema
    - OneCLI proxy reachable at gateway URL injected by host
    - Anthropic credentials resolvable through OneCLI for this agent_group_id
  STEPS:
    1. Compose SOUL:
       IF APEX_NANOCLAW_AWARE AND /workspace/CLAUDE.md exists THEN
         soul = compose_soul_from_nanoclaw(/workspace)
       ELSE
         soul = soul_registry.get(APEX_SOUL_ID)
       END IF
    2. Initialize apex.llm.pipeline with SOUL + telemetry sinks
    3. Initialize MCP host in-process, register @tool-decorated callables,
       attach external MCP servers declared by SOUL
    4. Open inbound.db (read-write) and outbound.db (read-write) via stdlib sqlite3
       (sqlite3.connect with journal_mode=DELETE)
    5. Spawn heartbeat thread: touch /workspace/.heartbeat every 5 seconds
    6. POLL LOOP:
       a. SELECT row FROM messages_in WHERE delivered_to_agent_at IS NULL
          ORDER BY (on_wake DESC, seq ASC) LIMIT 1
       b. IF no row, sleep 250ms, continue
       c. Mark row.delivered_to_agent_at = now
       d. Run AgentLoop(soul, row.content, history_window) which:
          - Assembles prompt with SOUL system + history + tool defs (cache-aware)
          - Calls apex.llm.pipeline.call(...) → response or tool_calls
          - If tool_calls: dispatch via MCP; loop until final answer or max_iterations
          - Persists every step to telemetry
       e. INSERT INTO messages_out (seq = next_odd_seq(), kind, content, ...)
       f. UPDATE session_state with new context window pointer
  POSTCONDITIONS:
    - Every inbound row has either a corresponding outbound row OR a recorded error row
    - Heartbeat path mtime within last 10 seconds while process is alive
    - Telemetry rows persisted for every LLM call

ERROR PATHS:
    - Provider 429: exponential backoff up to 3 retries; on final failure write
      error kind to outbound.messages_out + log to telemetry with error column
    - Output PII gate redacts: emit redacted content + telemetry gate_blocks JSON
    - Tool gate rejects arg: emit tool_error to agent loop, allow retry within
      max_iterations; if exhausted, deliver "I could not complete that action"
    - DB write failure: log + stop polling + let heartbeat lapse so host-sweep
      detects and restarts container
```

### Architecture (SPARC)

```
                       ┌──────────────────────────────────────┐
                       │   NanoClaw host (unchanged)          │
                       │   channels · router · delivery       │
                       │   container-runner · session-mgr     │
                       └─────────────┬────────────────────────┘
                                     │ writes inbound.db, reads outbound.db
                                     ▼
              ┌──────────────────────────────────────────────────┐
              │  Per-session container — image chosen by         │
              │  container_configs.backend:                      │
              │                                                  │
              │  ┌──────────────────┐    ┌────────────────────┐  │
              │  │ nanoclaw-agent   │ OR │  apex-agent        │  │
              │  │ (Bun, today)     │    │  (Python, new)     │  │
              │  │                  │    │                    │  │
              │  │ poll inbound.db  │    │ poll inbound.db    │  │
              │  │ Claude SDK loop  │    │ apex.agents loop   │  │
              │  │ write outbound   │    │ write outbound     │  │
              │  └──────────────────┘    └──────────┬─────────┘  │
              │                                     │            │
              └─────────────────────────────────────┼────────────┘
                                                    │
                                                    ▼
                              ┌──────────────────────────────────┐
                              │ apex.llm pipeline (in-process)   │
                              │ cache → gates → provider → judge │
                              │ → drift → telemetry              │
                              └──────────────────────────────────┘
```

**Modules**:

| Module | Responsibility | Depends on |
|--------|----------------|------------|
| `container/apex-agent/src/apex/runtime/__main__.py` | Poll inbound.db, run agent loop, write outbound.db, heartbeat | `apex.agents`, `apex.llm`, stdlib `sqlite3` |
| `container/apex-agent/src/apex/llm/pipeline.py` | Orchestrate gates, cache, provider call, judge, drift, telemetry per request | All `apex.llm.*` submodules |
| `container/apex-agent/src/apex/llm/providers/anthropic.py` | Anthropic SDK adapter with prompt-cache breakpoint insertion | `anthropic` Python SDK |
| `container/apex-agent/src/apex/llm/cache/anthropic_cache.py` | Resolve SOUL cache config into `cache_control` breakpoints (max 4) | `apex.llm.providers.base` |
| `container/apex-agent/src/apex/llm/telemetry/{sqlite,stdout,firestore,bigquery,otel}.py` | Pluggable telemetry sinks; SQLite + stdout default on, GCP optional | stdlib + optional GCP libs |
| `container/apex-agent/src/apex/llm/gates/*.py` | Injection, intent, output PII, tool-arg validators | `apex.llm.pipeline` |
| `container/apex-agent/src/apex/agents/runtime/loop.py` | Bounded agent loop; dispatches tool calls via MCP | `apex.mcp`, `apex.llm.pipeline` |
| `container/apex-agent/src/apex/agents/souls/{registry,builtin,nanoclaw_adapter}.py` | SOUL registry + built-ins + optional NanoClaw CLAUDE.md/skills bootstrap | `apex.agents.skills` |
| `container/apex-agent/src/apex/mcp/{server,client,registry}.py` | In-process MCP host + client; auto-register `@tool` callables | `mcp` Python SDK |
| `src/db/migrations/016-apex-backend.ts` | Add `backend` column to `container_configs` | NanoClaw migrations infra |
| `src/db/container-configs.ts` | Extend type + CRUD scalars to include `backend` | existing |
| `src/container-runner.ts` (modified) | Fork image + command on `containerConfig.backend` | existing |
| `src/install-slug.ts` (modified) | Add `getApexContainerImageBase()` mirroring nanoclaw variant | existing |
| `src/cli/resources/groups.ts` (modified) | `--backend` flag on `config update`; dual-section output on `config get` | existing |
| `src/claude-md-compose.ts` (modified) | Skip composition when `backend === 'apex'`; apex composes its own | existing |
| `container/apex-agent/Dockerfile` | Python base image, apex package install, entrypoint | python:3.11-slim base |
| `container/apex-agent/build.sh` | Mirror of `container/build.sh`; image tag derived from install-slug | `setup/lib/install-slug.sh` |

**Bounded contexts touched**:

- `src/db/` (NanoClaw migrations + `container-configs`) — owner: NanoClaw core
- `src/container-runner.ts` + `src/cli/resources/groups.ts` + `src/claude-md-compose.ts` — owner: NanoClaw core
- `container/apex-agent/` (NEW) — owner: Apex sub-team (solutions@getapexinsights.com)
- `container/build.sh` + `setup/container.ts` — owner: NanoClaw core (light touch for opt-in apex build)

### Data Model

NanoClaw central DB change (single column add via migration 016):

```
ALTER TABLE container_configs
  ADD COLUMN backend TEXT NOT NULL DEFAULT 'nanoclaw'
  CHECK (backend IN ('nanoclaw', 'apex'));
```

Per-session telemetry DB written by apex container (lives at `/workspace/.apex/telemetry.db`, one DB per session):

```
CREATE TABLE llm_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts TEXT NOT NULL,
  session_id TEXT NOT NULL,
  agent_group_id TEXT NOT NULL,
  feature TEXT,
  action TEXT,
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  input_tokens INTEGER,
  output_tokens INTEGER,
  cache_creation_input_tokens INTEGER,
  cache_read_input_tokens INTEGER,
  cost_usd REAL,
  latency_ms INTEGER,
  trace_id TEXT,
  gate_blocks TEXT,
  judge_score REAL,
  error TEXT
);
CREATE INDEX llm_events_session_ts ON llm_events(session_id, ts);
```

No schema changes to `inbound.db` or `outbound.db` — apex reads + writes the existing NanoClaw schema verbatim.

### API Surface

apex does not expose external HTTP endpoints in v0.1. The "API surface" is the SQL contract over the two session DBs plus the `ncl` CLI extensions.

| Verb | Path | Purpose | Auth |
|------|------|---------|------|
| SQL READ | `inbound.db` `messages_in` | apex polls undelivered messages (ordered by `on_wake DESC, seq ASC`) | container mount-scoped |
| SQL WRITE | `outbound.db` `messages_out` | apex writes agent responses using next odd-numbered seq | container mount-scoped |
| SQL WRITE | `outbound.db` `session_state` | apex updates context window pointer per turn | container mount-scoped |
| CLI | `ncl groups config update --backend <nanoclaw\|apex>` | Flip agent-group backend; persists to `container_configs.backend` | owner / global admin / scoped admin |
| CLI | `ncl groups config get --id <id>` | Returns `### NanoClaw composed CLAUDE.md` + `### Apex SOUL config` sections, labeled | member or admin |
| CLI | `ncl groups restart --id <id>` | Existing command; respawns container with new backend image after flip | owner / admin |

---

## What changes / What stays

| Area | Today | After this decision |
|------|-------|---------------------|
| Per-session agent code | `nanoclaw-agent` Bun container with Claude Agent SDK | Same OR `apex-agent` Python container, selectable per agent group |
| LLM call layer | Direct Anthropic SDK calls inside Bun agent-runner | `apex.llm` pipeline with gates + cache + judge + drift + telemetry (when backend=apex) |
| Tool definition | TypeScript MCP tools in `container/agent-runner/src/mcp-tools/` | TypeScript MCP tools (nanoclaw) **OR** Python `@tool` decorator that auto-registers as MCP server (apex) |
| Telemetry | NanoClaw host logs (`logs/nanoclaw.log`, `logs/nanoclaw.error.log`) | NanoClaw host logs **plus** per-session `.apex/telemetry.db` (when backend=apex); GCP sinks optional |
| Prompt caching | Default Claude SDK behavior, no explicit cache control | Apex provider adapter inserts up to 4 `cache_control` breakpoints from SOUL config; usage tokens recorded separately |
| Backend selection | Single image, single command | Per-agent-group flag (`container_configs.backend`) chooses image + command |
| Container build | `pnpm run setup` builds `nanoclaw-agent` only | Same default; opt-in `INSTALL_APEX=true` adds apex image build |

**Stays untouched** (load-bearing reassurance):

- All NanoClaw host code: `src/router.ts`, `src/delivery.ts`, `src/session-manager.ts`, `src/host-sweep.ts`, channel adapters under `src/channels/`
- `inbound.db` and `outbound.db` schemas; the odd/even seq parity convention; `journal_mode=DELETE` cross-mount visibility invariant
- Channel adapters (Slack, Telegram, WhatsApp, Discord, etc.) — apex never sees these directly
- OneCLI gateway integration (`src/onecli-approvals.ts`, `ensureAgent()` flow in `container-runner.ts`)
- Existing `nanoclaw-agent` image, Bun-based agent-runner, and its Claude SDK + MCP-tools tree
- Default behavior for existing installs: backend stays `nanoclaw`, no new toolchain forced

### Surface Matrix

> Per the project's `CLAUDE.md` Surface Parity Rule.

| Surface | Affected? | Notes |
|---------|-----------|-------|
| Admin desktop | N/A | NanoClaw has no web admin UI; configuration is `ncl` CLI only |
| Admin mobile | N/A | Same as above |
| Portal desktop | N/A | Not applicable; NanoClaw is a personal assistant host, not a customer-facing portal |
| Portal mobile | N/A | Same |
| Published site / live URL | N/A | Same |
| Funnel pages (if applicable) | N/A | Same |
| Quiz runtime (if applicable) | N/A | Same |
| Super admin only | yes | The `--backend` flag in `ncl groups config update` requires owner or admin role per `src/command-gate.ts` policy; backed by `user_roles` table |
| Org owner / admin / member | partial | Owner and global admin can flip any group; scoped admin can flip groups they administer; unprivileged members cannot |
| Venue manager / staff | N/A | NanoClaw is not multi-tenant per venue |

### Venue Impact

N/A — platform-wide change to NanoClaw's container infrastructure; no per-venue divergence.

---

## Operator workflows

### Workflow A: Flip an existing agent group to Apex

1. Identify the agent group: `ncl groups list`
2. Flip the backend: `ncl groups config update --id <group-id> --backend apex`
3. Optionally enable bootstrap from existing CLAUDE.md/skills: `ncl groups config update --id <group-id> --env APEX_NANOCLAW_AWARE=true`
4. Trigger a clean swap: `ncl groups restart --id <group-id>` — kills the current container; on next message the host spawns the apex image
5. Send a message via the wired channel; confirm response delivered. Inspect `ncl sessions list` for an apex-tagged session and `pnpm exec tsx scripts/q.ts data/v2-sessions/<group-id>/<session-id>/outbound.db "SELECT * FROM messages_out ORDER BY seq DESC LIMIT 5"` for the response row

### Workflow B: Flip back to NanoClaw

1. `ncl groups config update --id <group-id> --backend nanoclaw`
2. `ncl groups restart --id <group-id>`
3. Subsequent messages handled by `nanoclaw-agent` again. Conversation history in `inbound.db` / `outbound.db` is preserved; both backends read the same rows

---

## Open Questions

**Q1.** Should the apex Docker image be developed in-tree (`container/apex-agent/`) or as a sibling repo from day one?
- **Default**: in-tree for v0.1 (tight dev loop, shared build pipeline, single PR review surface); extract to a sibling repo once the two-DB contract is proven and stable.
- **Decision**: In-tree under `container/apex-agent/` for the full NanoClaw integration cycle (Phases 1–5 of this plan). Extract to a sibling repo at the apex v1.0 / public OSS launch (a separate ADR will own that move).

**Q2.** Should `INSTALL_APEX=true` trigger the apex image build during the normal `pnpm run setup`, or require a separate `pnpm run setup:apex` step?
- **Default**: trigger during normal `setup` when the env var is set; print a clear opt-in banner so the user knows what is happening.
- **Decision**: Trigger inside `setup/container.ts` when `INSTALL_APEX=true` is set in `.env`. Print `Building apex-agent image (opt-in via INSTALL_APEX=true)...` so the operator sees the extra step. Default remains off.

**Q3.** Should apex's tool catalog be auto-exposed as an MCP server reachable by `nanoclaw-agent` containers, or kept private to apex containers in v0.1?
- **Default**: keep private in v0.1; cross-backend MCP-sharing is a clear v0.2 win but raises lifecycle questions (whose container hosts the shared MCP server? who restarts it?).
- **Decision**: Keep private to apex containers in v0.1. A future ADR will cover the cross-backend tool registry once we have one tool-rich apex deployment in production.

**Q4.** Should the NanoClaw-aware bootstrap parse NanoClaw skills as structured apex Skills, or treat them as a flat system-prompt addendum?
- **Default**: flat system-prompt addendum in v0.1 (cheap, behavior-preserving, no schema-mapping risk); proper `SKILL.md` → `apex.Skill` translation deferred until apex Skill format is stable.
- **Decision**: Flat addendum in v0.1. Concatenate each `SKILL.md` body into the SOUL's system prompt, prefixed with the skill name. Document the translation gap in `container/apex-agent/docs/nanoclaw-bootstrap.md`.

**Q5.** How does apex handle NanoClaw's `messages_in.on_wake` semantic (wake messages picked up only by a fresh container's first poll iteration)?
- **Default**: mirror exactly. First poll iteration after spawn must process any `on_wake = 1` row before regular queue; thereafter, ignore the flag.
- **Decision**: Mirror NanoClaw's exact ordering — `ORDER BY on_wake DESC, seq ASC`. First-iteration logic in `apex/runtime/poll_loop.py` references the NanoClaw counterpart at `container/agent-runner/src/db/messages-in.ts:getPendingMessages` in a comment block so the invariant is discoverable.

---

## Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Schema drift between NanoClaw `inbound.db`/`outbound.db` and apex's reader | high | Apex generates its DB schema view from NanoClaw migrations at build time (parses SQL from `src/db/migrations/`); Phase 2 test asserts read fidelity against a NanoClaw-produced sample DB |
| Heartbeat lifecycle differences between Bun and Python (different signal handling, GIL pauses) | medium | Apex uses identical heartbeat path (`/workspace/.heartbeat`) and 5-second touch interval; mirror NanoClaw's `host-sweep` grace period; Phase 3 integration test forces a Python GC pause and confirms heartbeat continuity |
| OneCLI integration breaks when the image differs (wrong CA cert path, wrong proxy env vars) | medium | Apex Dockerfile installs the same `/etc/ssl/certs/onecli-ca.crt` location; `container-runner.ts` injects identical proxy env vars regardless of backend; smoke-tested in Phase 3 |
| Build pipeline doubles (Bun + Python toolchain) inflates setup time | low | Default `INSTALL_APEX=false` keeps host install lean; apex build only runs when explicitly opted in |
| Telemetry SQLite write contention between concurrent agent loops on same host | low | One `telemetry.db` per session (lives at `/workspace/.apex/`), not shared across containers; SQLite WAL/DELETE behavior tested in Phase 3 |
| MCP server registration race on cold start (poll loop starts before tools register) | medium | Block first poll iteration until MCP registration future resolves; assert in Phase 4 test |
| Anthropic `cache_control` breakpoint count exceeds the 4-limit | low | Provider adapter validates breakpoint count at SOUL compile time; fail loud with a clear error, not silent truncation |
| Two SOUL/skill catalogs drift apart (NanoClaw skills vs apex SOULs) | medium | `APEX_NANOCLAW_AWARE=true` bootstrap path lets users keep one source of truth (NanoClaw); operators choosing native apex SOULs accept the maintenance burden |

**Debugging guarantees** (adapted from the project's CLAUDE.md debugging culture):

- Before changing `container_configs.ts` or `container-runner.ts`, we will inspect the live `data/v2.db` schema and current row data with `pnpm exec tsx scripts/q.ts data/v2.db "SELECT * FROM container_configs LIMIT 5"`.
- Before claiming the swap works end-to-end, we will run a side-by-side comparison: same input message, same channel, once with `backend=nanoclaw` and once with `backend=apex`, then diff `outbound.db` `messages_out` rows.
- For session DB cross-mount visibility regressions, we will check `journal_mode` first (must remain `DELETE`) before debugging payload contents — this is the NanoClaw two-DB invariant.

**Rollback story**: A single SQL update reverts every flipped group: `UPDATE container_configs SET backend = 'nanoclaw'`. Migration 016 ships with a `down()` that drops the column. In-flight apex containers continue until they die (heartbeat lapse or `ncl groups restart`); fresh sessions respawn with `nanoclaw-agent`. No data migration required — both backends read the same session DBs.

---

## Consequences

### Positive

- Adds a swappable alternative LLM pipeline without modifying NanoClaw host code or destabilizing existing agent groups.
- Centralizes safety + observability + caching + budget in one Python place, reusable from any future Python agent context (web app, cron job, sidecar).
- Validates the "portable agent backend" thesis (Docker image speaks two-DB protocol; runs anywhere NanoClaw runs).
- Lays foundation for apex v1.0 OSS release without coupling that release to NanoClaw's lifecycle.

### Negative

- Adds a second container image and toolchain (Python + uv/pip) to maintain alongside Bun.
- Splits future agent feature work between two stacks (`nanoclaw-agent` Bun and `apex-agent` Python).
- Two SOUL/skill catalogs to keep in sync if both backends see active use; mitigated by `APEX_NANOCLAW_AWARE` bootstrap but not eliminated.
- New contributors must understand both stacks to debug a session from channel-in to LLM-out.

### Neutral / Worth noting

- The two-DB contract becomes a normalized agent-backend interface: future alternative backends (Rust, Go, an Atomic agent-runner) plug in the same way. Adds optionality without complexity per agent group.
- Apex's prompt-cache and MCP investment can be backported into `nanoclaw-agent` later (separate ADR) — this decision does not foreclose that.

---

## Methodology

- **Sub-agents invoked**: none — decision authored interactively with the user through structured AskUserQuestion rounds; multiple architecture options (Option A = apex as gateway, Option B = apex as full backend, parallel deployment) compared before locking.
- **Sources consulted**:
  - NanoClaw codebase: `src/container-runner.ts`, `src/db/migrations/*.ts`, `src/install-slug.ts`, `container/build.sh`, `container/agent-runner/src/poll-loop.ts`, `CLAUDE.md`, `docs/architecture.md`
  - Apex Insights design notes at `docs/potential-mods/apex.llm` (originating two-package design)
  - Anthropic prompt-caching documentation (cache_control, cache_creation/read tokens)
  - MCP Python SDK reference (`mcp` package, stdio transport)
- **Comparable platforms surveyed**:
  - LiteLLM (provider abstraction only — rejected as too thin for apex's safety/judge/drift scope)
  - Langfuse, Helicone (observability-only — rejected as out-of-band, not call-layer)
  - Google Agent Development Kit, OpenAI Agents SDK, Pydantic AI, LangGraph (agent runtimes — reference points for SOUL/Skill/Tool primitives; none provide cohesive call-layer + runtime + multi-channel)
  - NanoClaw's existing OpenCode provider precedent (validates that per-group backend selection is already a supported axis of variation)
- **Counter-evidence considered**:
  - "Just enhance nanoclaw-agent in place" — rejected because it forces Bun for every future LLM feature and bloats a single image
  - "Make apex a pure HTTP gateway, no agent loop" — rejected because it leaves agent reasoning trapped in nanoclaw-agent (Bun/TS); SOULs/Skills/memory would have to be ported to TS or duplicated
  - "Replace nanoclaw-agent entirely with apex" — rejected because the migration cost is large with no incremental benefit and abandons the existing Bun + Claude SDK investment

---

## Industry evidence

| Platform | What they do | Source (URL + date) |
|----------|--------------|---------------------|
| Google ADK (Agent Development Kit) | Open-source Python agent framework; per-agent definitions, deployed on Vertex AI | https://google.github.io/adk-docs/ (Cloud Next 2025 launch) |
| OpenAI Agents SDK | Python multi-agent runtime; handoffs, tools, sessions; successor to Swarm | https://github.com/openai/openai-agents-python (2025) |
| Pydantic AI | Type-safe Python agent framework; provider-agnostic; structured outputs | https://ai.pydantic.dev/ (2024–2025) |
| LiteLLM | Provider abstraction + budget + retries; lacks runtime/agent layer | https://docs.litellm.ai/ (active 2024–2026) |
| Langfuse | Open-source LLM observability + tracing + evals; out-of-band, not call-layer | https://langfuse.com/docs (active 2024–2026) |

---

## Approval

When this ADR's status flips to **Accepted**:

1. The companion plan is the source of truth for execution
2. `promote-to-loop.sh apex-agent-backend` initializes dev-plan-loop state
3. `/loop iterate the next phase of .claude/plans/apex-agent-backend-plan.md` starts execution

---

## Changelog

- 2026-05-21 — Drafted and accepted by solutions@getapexinsights.com via `decide-plan-loop` skill (single-author flow; decisions captured through 4 AskUserQuestion rounds in the originating conversation)
