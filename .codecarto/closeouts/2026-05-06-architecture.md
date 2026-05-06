# Closeout — 2026-05-06 — architecture

**Phase:** architecture (Phase 1 of pipeline-full-with-deep-audit)
**Project:** hermes-agent
**Branch:** claude/codecarto-hermes-analysis-abvQm
**Session role:** implementing

## What was done

Produced the architecture map for hermes-agent (~200K LOC Python AI-agent platform). Read 18 files (well under the 30-file budget): pyproject.toml, README.md, AGENTS.md, Dockerfile, docker-compose.yml, .env.example, setup-hermes.sh, run_agent.py header, hermes_cli/main.py header, gateway/run.py header, mcp_serve.py header, acp_adapter/entry.py header, tests/conftest.py, .github/workflows/ listing, plus directory listings for agent/, tools/, gateway/, hermes_cli/, skills/.

Committed in three increments to survive any partial-write timeout:
1. `architecture-map.md` draft (without validation block)
2. Five secondary outputs (public-surfaces, runtime-lifecycle, state-and-storage, build-and-deploy, config-model) in one push
3. Validation block appended to `architecture-map.md`

## Outputs

- `findings/architecture/architecture-map.md` — 34,915 bytes (with validation block)
- `findings/public-surfaces/public-surfaces.md` — ~5.8 KB
- `findings/runtime-lifecycle/runtime-lifecycle.md` — ~5.5 KB
- `findings/state-and-storage/state-and-storage.md` — ~6.5 KB
- `findings/build-and-deploy/build-and-deploy.md` — ~6.0 KB
- `findings/config-model/config-model.md` — ~5.0 KB

## Validation

All five completion criteria PASS. No PARTIAL or FAIL rows. Gaps routed to `open_questions` (5 entries) or `carry_forward` (11 entries) per VALIDATE.md `target_phase` rubric.

## Decisions Beyond Prompt

- **D1: Did NOT read the four largest source files.** `gateway/run.py` (634 KB), `hermes_cli/main.py` (390 KB), `gateway/platforms/feishu.py` (198 KB), `gateway/platforms/yuanbao.py` (186 KB), `gateway/platforms/base.py` (131 KB), `gateway/platforms/api_server.py` (125 KB), `tools/mcp_tool.py` (127 KB), and `cli.py` / `model_tools.py` not opened. AGENTS.md, README, manifests, CI workflows, and directory listings provided enough structural signal. Deeper reads deferred via open_questions / carry_forward to phases that need them (protocols for gateway/run.py wire formats; defect-scan-semantic for the suspected concurrency hazards).
- **D2: Set `current_phase` to a comma-separated string** (`"defect-scan-mechanical, contracts, protocols (parallel)"`) rather than picking one phase, to make the parallelism explicit. The pipeline yaml sequences these post-architecture, but their dependency declarations only require architecture, so concurrent execution is safe.

## Proposed Conventions

- **Conv-1: Pre-loaded recon for implementing-session prompts.** When orchestrator-spawned implementing sessions analyze a large codebase, the orchestrator should pass structural recon (entry points, dir layout, manifests) directly in the prompt rather than asking the implementing session to re-discover. Cuts read budget by 30–50% and prevents stream-idle timeouts on slow-search phases.
- **Conv-2: Incremental commits per phase.** Implementing sessions should push primary draft → secondaries → validation block as three separate commits rather than batching everything into one final push. Survives partial-write timeouts and matches the framework's read-write segregation (primary owns load-bearing claims, secondaries own enumerated detail).

## Citations

- Primary output: `.codecarto/findings/architecture/architecture-map.md`
- Pipeline: `.codecarto/workflow/pipeline-full-with-deep-audit.yaml` (architecture phase)
- Validation rubric: `.codecarto/workflow/VALIDATE.md`
- Open questions: arch-OQ1 through arch-OQ5 (see status.yaml)
- Carry-forward: arch-CF1 through arch-CF11 (see status.yaml)

## Next phases

Defect-scan-mechanical, contracts, and protocols are now eligible (each depends only on architecture). Orchestrator will fan them out in parallel.
