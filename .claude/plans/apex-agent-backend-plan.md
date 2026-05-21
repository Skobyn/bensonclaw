# Apex Agent Backend for NanoClaw — Development Plan

> **ADR**: [`.claude/tasks/apex-agent-backend-adr.md`](../tasks/apex-agent-backend-adr.md)
> **Goal**: Add Apex (Python) as a swappable per-agent-group container backend that speaks NanoClaw's two-DB protocol, with MCP and Anthropic prompt cache first-class, telemetry portable by default, and one-flag flip between `nanoclaw` and `apex`.
> **Owner**: solutions@getapexinsights.com
> **Started**: 2026-05-21
> **Target**: TBD (single-operator iteration via `/loop`)

## Design Intent

Apex sits beside NanoClaw, not on top of it. NanoClaw remains the channel + session + container host. Apex is a parallel agent backend, selected per agent group via `container_configs.backend`, packaged as a Docker image that polls `inbound.db` and writes `outbound.db` exactly like `nanoclaw-agent` does today. The host code's only change is one fork in `container-runner.ts` that chooses image + command based on the new column.

Apex centralizes the LLM pipeline (gates, prompt cache, judge, drift, telemetry, budget) so future Python LLM work — cron jobs, sidecars, in-app calls — reuses the same primitives. MCP-native tool dispatch and Anthropic prompt caching are first-class in v0.1. Telemetry defaults to local SQLite + stdout; GCP sinks (Firestore, BigQuery, OTel) activate only when credentials are present. Ollama provider in Phase 5 validates "portable framework" — the same apex image runs on a laptop with no cloud dependencies.

**Bounded contexts**:

- `src/db/`, `src/container-runner.ts`, `src/cli/`, `src/claude-md-compose.ts` (NanoClaw core; light touch — single column + single spawn fork + CLI surface)
- `container/apex-agent/` (NEW — Python container, owned by Apex sub-team)
- `setup/container.ts` + `container/build.sh` (light touch for opt-in build trigger)

**Non-negotiable constraints**:

- Must not modify NanoClaw `inbound.db` / `outbound.db` schemas or the odd/even seq parity convention
- Must not require Python on the host — apex builds and runs only inside its own container
- Must reuse OneCLI gateway for secrets (no direct env-var credentials in apex containers)
- Default install behavior unchanged: backend defaults to `nanoclaw`, apex image not built unless `INSTALL_APEX=true`
- Must honor `journal_mode=DELETE` on session DBs (NanoClaw cross-mount visibility invariant)

---

## Execution Strategy

> **Default per phase**: hierarchical-mesh, 4–5 agents tuned per task (queen + specialists)
> **Override per task** via the `Swarm:` line on each checkbox.

### Swarm directives (per task)

| Directive | Meaning |
|-----------|---------|
| `Swarm: single [<agent-type>]` | One `Agent` tool invocation; orchestrator picks subagent_type |
| `Swarm: multi <count> [<t1>, <t2>, ...]` | N parallel `Agent` calls **in one message** |
| `Swarm: hierarchical <count> [<t1>, <t2>, ...]` | `mcp__claude-flow__swarm_init` + N spawns, queen-led |
| `Swarm: mesh <count> [<t1>, <t2>, ...]` | Peer-to-peer mesh topology, no queen |

### Approval gates (between phases)

| Gate tag | Behavior |
|----------|----------|
| `gate:auto` (in brackets) | Orchestrator runs the Acceptance check; advances on pass, halts on fail |
| `gate:human` (in brackets) | Orchestrator halts; user types the approval phrase from Acceptance |
| `gate:partner:<email>` (in brackets) | Orchestrator writes inbox item to `<email>`, halts until consumed |

All gates in this plan are `auto` or `human` — solo-operator flow, no partner sign-off needed.

---

## Phases

> **Task format** (parsed by dev-plan-loop `iterate.sh`):
> ```
> - [ ] **Phase X.Y** [tag1][tag2] Imperative task title
>   - Acceptance: <runnable check>
>   - Swarm: <directive>
>   - Blocked-by: phase-X.Y
> ```

---

### Phase 1 — Specification (SPARC)

Lock the integration surface against NanoClaw before writing any container code. Produce the artifacts later phases test against.

- [ ] **Phase 1.1** [research][docs] Survey NanoClaw integration points and write findings
  - Acceptance: `test -f docs/research/apex-agent-backend.md && python3 -c "import sys; sys.exit(0 if len(open('docs/research/apex-agent-backend.md').read().split()) >= 500 else 1)"`
  - Swarm: single [researcher]

- [ ] **Phase 1.2** [docs] Author the apex-vs-nanoclaw acceptance test matrix
  - Acceptance: `test -f docs/apex-agent-backend/test-matrix.md && grep -q 'backend=apex' docs/apex-agent-backend/test-matrix.md && grep -q 'backend=nanoclaw' docs/apex-agent-backend/test-matrix.md`
  - Swarm: single [analyst]
  - Blocked-by: phase-1.1

- [ ] **Phase 1.3** [docs] Confirm ADR-0001 is `Accepted`
  - Acceptance: `grep -q '^\*\*Status\*\*: Accepted' .claude/tasks/apex-agent-backend-adr.md`
  - Swarm: single [reviewer]

- [ ] **Gate 1→2** [gate:human] Specification sign-off — confirm scope and acceptance matrix before any code lands
  - Acceptance: user types `approve gate-1-2` in chat
  - Blocked-by: phase-1.3

---

### Phase 2 — Pseudocode (SPARC)

Scaffold the integration surface and an echo-only apex runtime. By the end of this phase, flipping `backend=apex` and sending a message yields an echo response delivered back through the same channel. No LLM call yet — this is the swap-works test.

- [ ] **Phase 2.1** [backend][infra] Migration 016 — add `backend` column to `container_configs`
  - Acceptance: `pnpm run dev > /tmp/migr.log 2>&1 & sleep 4; pnpm exec tsx scripts/q.ts data/v2.db "PRAGMA table_info(container_configs)" | grep -q backend && pkill -f "pnpm run dev"`
  - Swarm: hierarchical 3 [architect, coder, reviewer]
  - Blocked-by: gate-1-2

- [ ] **Phase 2.2** [backend] Extend `src/db/container-configs.ts` type + CRUD scalars to include `backend`
  - Acceptance: `grep -q "backend" src/db/container-configs.ts && pnpm run build`
  - Swarm: single [coder]
  - Blocked-by: phase-2.1

- [ ] **Phase 2.3** [backend][infra] Add `getApexContainerImageBase()` to `src/install-slug.ts`
  - Acceptance: `grep -q getApexContainerImageBase src/install-slug.ts && pnpm run build`
  - Swarm: single [coder]
  - Blocked-by: phase-2.1

- [ ] **Phase 2.4** [backend] Fork spawn logic in `src/container-runner.ts` — pick image + command based on `containerConfig.backend`
  - Acceptance: `grep -q "backend === 'apex'" src/container-runner.ts && pnpm run build && pnpm test -- src/container-runner.test.ts`
  - Swarm: hierarchical 3 [architect, coder, tester]
  - Blocked-by: phase-2.2, phase-2.3

- [ ] **Phase 2.5** [backend] Add `--backend` flag to `ncl groups config update` in `src/cli/resources/groups.ts`
  - Acceptance: `pnpm exec tsx src/cli/main.ts groups config update --help 2>&1 | grep -q -- --backend`
  - Swarm: single [coder]
  - Blocked-by: phase-2.2

- [ ] **Phase 2.6** [backend] `ncl groups config get` outputs labeled `### NanoClaw composed CLAUDE.md` AND `### Apex SOUL config` sections regardless of active backend
  - Acceptance: `pnpm test -- src/cli/resources/groups.test.ts -t "config get shows both sections"`
  - Swarm: hierarchical 3 [coder, tester, reviewer]
  - Blocked-by: phase-2.5

- [ ] **Phase 2.7** [infra] Author `container/apex-agent/Dockerfile` + `build.sh` (image tag derived from install-slug, mirrors `container/build.sh`)
  - Acceptance: `bash container/apex-agent/build.sh && docker image ls | grep -q apex-agent-v2`
  - Swarm: hierarchical 3 [architect, infra, reviewer]
  - Blocked-by: phase-2.4

- [ ] **Phase 2.8** [backend] Echo-only Python runtime at `container/apex-agent/src/apex/runtime/__main__.py` — poll `inbound.db`, echo each message to `outbound.db` with odd seq, touch heartbeat
  - Acceptance: `cd container/apex-agent && python -m pytest tests/unit/test_runtime_echo.py -v`
  - Swarm: hierarchical 4 [architect, coder, tester, reviewer]
  - Blocked-by: phase-2.7

- [ ] **Phase 2.9** [backend] Opt-in apex image build in `setup/container.ts` when `INSTALL_APEX=true` in `.env`
  - Acceptance: `INSTALL_APEX=true pnpm exec tsx setup/container.ts --dry-run 2>&1 | grep -q "apex-agent"`
  - Swarm: single [coder]
  - Blocked-by: phase-2.7

- [ ] **Phase 2.10** [tests] Integration test — flip backend, send message, verify echo round-trip via session DBs
  - Acceptance: `pnpm test -- src/tests/integration/apex-backend-swap.test.ts`
  - Swarm: hierarchical 3 [tester, coder, reviewer]
  - Blocked-by: phase-2.8, phase-2.9

- [ ] **Gate 2→3** [gate:auto] Swap works end-to-end with echo runtime
  - Acceptance: `pnpm run build && pnpm test && cd container/apex-agent && python -m pytest tests/unit/`
  - Blocked-by: phase-2.10

---

### Phase 3 — Architecture (SPARC)

Wire the real LLM call path. By the end of this phase, flipping `backend=apex` produces actual Claude responses with prompt-cache breakpoints recorded in telemetry.

- [ ] **Phase 3.1** [backend] `apex/llm/providers/anthropic.py` — Anthropic SDK adapter with cache_control breakpoint insertion (max 4)
  - Acceptance: `cd container/apex-agent && python -m pytest tests/llm/providers/test_anthropic_cache.py -v`
  - Swarm: hierarchical 4 [architect, coder, tester, reviewer]
  - Blocked-by: gate-2-3

- [ ] **Phase 3.2** [backend] `apex/llm/pipeline.py` — orchestrate cache → provider → telemetry per request (gates land in Phase 4)
  - Acceptance: `cd container/apex-agent && python -m pytest tests/llm/test_pipeline.py -v`
  - Swarm: hierarchical 3 [architect, coder, tester]
  - Blocked-by: phase-3.1

- [ ] **Phase 3.3** [backend] `apex/llm/telemetry/sqlite.py` and `apex/llm/telemetry/stdout.py` — default-on sinks, write to `/workspace/.apex/telemetry.db` and process stdout
  - Acceptance: `cd container/apex-agent && python -m pytest tests/llm/telemetry/ -v`
  - Swarm: hierarchical 3 [coder, tester, reviewer]
  - Blocked-by: phase-3.2

- [ ] **Phase 3.4** [backend][security] OneCLI proxy integration — apex container honors injected proxy env + CA cert mount; Anthropic SDK requests route through gateway
  - Acceptance: `cd container/apex-agent && python -m pytest tests/integration/test_onecli_proxy.py -v`
  - Swarm: hierarchical 4 [security-architect, coder, tester, reviewer]
  - Blocked-by: phase-3.1

- [ ] **Phase 3.5** [backend] Replace echo runtime with pipeline-backed loop — apex now calls Claude and writes responses to `outbound.db`
  - Acceptance: `cd container/apex-agent && python -m pytest tests/runtime/test_pipeline_loop.py -v`
  - Swarm: hierarchical 4 [architect, coder, tester, reviewer]
  - Blocked-by: phase-3.2, phase-3.3, phase-3.4

- [ ] **Phase 3.6** [tests] Host-side integration — flip backend, send message, verify real Claude response delivered + cache row present in telemetry.db
  - Acceptance: `pnpm test -- src/tests/integration/apex-backend-llm.test.ts`
  - Swarm: hierarchical 3 [tester, coder, reviewer]
  - Blocked-by: phase-3.5

- [ ] **Gate 3→4** [gate:auto] Real LLM round-trip green; cache token columns populated
  - Acceptance: `pnpm test && cd container/apex-agent && python -m pytest tests/ && pnpm exec tsx scripts/q.ts /tmp/apex-test/.apex/telemetry.db "SELECT cache_read_input_tokens FROM llm_events LIMIT 1"`
  - Blocked-by: phase-3.6

---

### Phase 4 — Refinement (SPARC, TDD)

Make every component real. Agent loop with MCP tool dispatch, gates, judge, drift, budget, and NanoClaw-aware bootstrap.

- [ ] **Phase 4.1** [backend] `apex/mcp/{server,client,registry}.py` + `apex/agents/tools/decorator.py` — `@tool` callables auto-register as MCP tools via in-process MCP host
  - Acceptance: `cd container/apex-agent && python -m pytest tests/mcp/ -v`
  - Swarm: hierarchical 4 [architect, coder, tester, reviewer]
  - Blocked-by: gate-3-4

- [ ] **Phase 4.2** [backend] `apex/agents/runtime/loop.py` — bounded tool-use loop; all tool dispatch via MCP; max_iterations enforced
  - Acceptance: `cd container/apex-agent && python -m pytest tests/agents/runtime/test_loop.py -v`
  - Swarm: hierarchical 4 [architect, coder, tester, reviewer]
  - Blocked-by: phase-4.1

- [ ] **Phase 4.3** [backend] `apex/agents/souls/{registry,builtin}.py` + built-in `assistant-base` SOUL with cache config
  - Acceptance: `cd container/apex-agent && python -m pytest tests/agents/souls/ -v`
  - Swarm: hierarchical 3 [coder, tester, reviewer]
  - Blocked-by: phase-4.1

- [ ] **Phase 4.4** [backend] `apex/agents/souls/nanoclaw_adapter.py` — compose SOUL from `/workspace/CLAUDE.md` + flat skill addenda from `/workspace/skills/*/SKILL.md`
  - Acceptance: `cd container/apex-agent && python -m pytest tests/agents/souls/test_nanoclaw_adapter.py -v`
  - Swarm: hierarchical 3 [coder, tester, reviewer]
  - Blocked-by: phase-4.3

- [ ] **Phase 4.5** [backend] Runtime honors `APEX_NANOCLAW_AWARE` env flag — if true and CLAUDE.md present, compose from NanoClaw; else use registered SOUL
  - Acceptance: `cd container/apex-agent && python -m pytest tests/runtime/test_bootstrap_flag.py -v`
  - Swarm: hierarchical 3 [coder, tester, reviewer]
  - Blocked-by: phase-4.4

- [ ] **Phase 4.6** [backend][security] `apex/llm/gates/` — injection scanner, intent scope, output PII, tool-arg validator; all policy-driven
  - Acceptance: `cd container/apex-agent && python -m pytest tests/llm/gates/ -v`
  - Swarm: hierarchical 4 [security-architect, security-auditor, coder, tester]
  - Blocked-by: phase-4.2

- [ ] **Phase 4.7** [backend] `apex/llm/{judge,drift,budget}.py` — async judge worker, four drift detectors, monthly cap enforcement
  - Acceptance: `cd container/apex-agent && python -m pytest tests/llm/test_judge.py tests/llm/test_drift.py tests/llm/test_budget.py -v`
  - Swarm: hierarchical 4 [architect, coder, tester, reviewer]
  - Blocked-by: phase-4.2

- [ ] **Phase 4.8** [tests] Integration — flip existing NanoClaw agent group to apex with `APEX_NANOCLAW_AWARE=true`, verify behavioral parity on smoke prompt
  - Acceptance: `pnpm test -- src/tests/integration/apex-nanoclaw-bootstrap.test.ts`
  - Swarm: hierarchical 3 [tester, coder, reviewer]
  - Blocked-by: phase-4.5, phase-4.6

- [ ] **Gate 4→5** [gate:auto] Gates + agent loop + bootstrap green
  - Acceptance: `pnpm test && cd container/apex-agent && python -m pytest tests/`
  - Blocked-by: phase-4.7, phase-4.8

---

### Phase 5 — Completion (SPARC)

Multi-provider, portable validation, docs, CI. Apex usable on a laptop with local models and no cloud dependencies.

- [ ] **Phase 5.1** [backend] `apex/llm/providers/openai.py` + `apex/llm/providers/gemini.py`
  - Acceptance: `cd container/apex-agent && python -m pytest tests/llm/providers/test_openai.py tests/llm/providers/test_gemini.py -v`
  - Swarm: hierarchical 3 [coder, tester, reviewer]
  - Blocked-by: gate-4-5

- [ ] **Phase 5.2** [backend] `apex/llm/providers/ollama.py` — local-model provider; validates portable-framework thesis
  - Acceptance: `cd container/apex-agent && python -m pytest tests/llm/providers/test_ollama.py -v`
  - Swarm: hierarchical 3 [coder, tester, reviewer]
  - Blocked-by: gate-4-5

- [ ] **Phase 5.3** [tests] Full integration suite — host tests + container tests + Ollama demo (laptop, no GCP)
  - Acceptance: `pnpm test && cd container/apex-agent && python -m pytest tests/ -v`
  - Swarm: hierarchical 3 [tester, debugger, reviewer]
  - Blocked-by: phase-5.1, phase-5.2

- [ ] **Phase 5.4** [infra] CI workflow `.github/workflows/apex-agent.yml` — builds apex image + runs pytest + runs host integration test in a matrix with `backend=apex`
  - Acceptance: `test -f .github/workflows/apex-agent.yml && grep -q pytest .github/workflows/apex-agent.yml && grep -q "backend.*apex" .github/workflows/apex-agent.yml`
  - Swarm: hierarchical 3 [cicd-engineer, infra, reviewer]
  - Blocked-by: phase-5.3

- [ ] **Phase 5.5** [docs] `container/apex-agent/README.md`, `container/apex-agent/docs/nanoclaw-bootstrap.md`, plus CLAUDE.md mention of the new backend axis
  - Acceptance: `test -f container/apex-agent/README.md && test -f container/apex-agent/docs/nanoclaw-bootstrap.md && grep -q apex-agent CLAUDE.md`
  - Swarm: single [docs-writer]
  - Blocked-by: phase-5.3

- [ ] **Phase 5.6** [docs] Flip ADR-0001 status to `Implemented`
  - Acceptance: `grep -q '^\*\*Status\*\*: Implemented' .claude/tasks/apex-agent-backend-adr.md`
  - Swarm: single [coder]
  - Blocked-by: phase-5.4, phase-5.5

- [ ] **Gate 5→done** [gate:human] Ship sign-off — apex backend ready for daily use
  - Acceptance: user types `approve ship` in chat
  - Blocked-by: phase-5.6

---

## Out-of-scope

> Explicit non-goals so the swarm does not drift.

- Replacing `nanoclaw-agent`. Both backends remain first-class; selection is per agent group.
- Cross-backend tool sharing (apex tools callable from nanoclaw-agent or vice versa). Deferred to a future ADR per Open Question Q3.
- Structured `SKILL.md` → `apex.Skill` translation. v0.1 ships flat system-prompt addenda per Open Question Q4.
- New host-side IPC, REST, or gRPC surfaces. Two-DB protocol is the only contract.
- Admin web UI for backend selection. `ncl` CLI is the sole control surface in v0.1.
- Vision, multimodal, embeddings, streaming token-by-token, semantic response cache. All deferred to a post-v0.1 ADR.
- Public OSS release of the apex package. In-tree under `container/apex-agent/` for the full integration cycle; extraction to a sibling repo is a separate decision.

## Open questions (escalations)

> If anything during execution requires a human decision, write it here with an `@OWNER` tag.

- (none yet)

---

## Continuity layer suggestions

Run alongside `/loop`:

```bash
# Nightly progress audit
/schedule "0 2 * * *" .claude/skills/dev-plan-loop/scripts/audit.sh .claude/plans/apex-agent-backend-plan.md

# Weekly architecture drift review
/schedule "0 9 * * 1" .claude/skills/dev-plan-loop/scripts/architecture-review.sh .claude/plans/apex-agent-backend-plan.md
```

---

## Status checks

```bash
# Current state
.claude/skills/decide-plan-loop/scripts/status.sh apex-agent-backend

# Evaluate a single gate without /loop running
.claude/skills/decide-plan-loop/scripts/gate.sh .claude/plans/apex-agent-backend-plan.md gate-2-3
```
