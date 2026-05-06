# Architecture Map

<!-- Phase 1 primary output. Validation block appended in a follow-up commit. -->

## System Intent

Hermes Agent is a self-improving, multi-surface AI agent platform built by Nous Research (MIT-licensed, Python 3.11+, ~200K LOC, version 0.12.0). It is a single, opinionated *agent shell* that wraps a synchronous tool-calling loop around any OpenAI-compatible LLM endpoint and exposes that loop through three primary user surfaces: an interactive terminal CLI (`hermes`), an Ink/React TUI (`hermes --tui`), and a multi-platform messaging gateway (`hermes-agent` / `hermes gateway`) that bridges Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Email, SMS, Mattermost, Teams, and several Chinese platforms (Feishu, DingTalk, WeCom, Weixin, QQ, Yuanbao). The system is differentiated by a closed *learning loop* — agent-curated cross-session memory, periodic memory nudges, autonomous skill creation, FTS5 session search with LLM summarization, and Honcho dialectic user modeling — plus a built-in cron scheduler, six pluggable terminal backends (local/docker/ssh/modal/daytona/singularity), an Atropos/Tinker RL training surface, an ACP server for IDE integration (VS Code/Zed/JetBrains), an MCP server, and an OpenAI-compatible API server. The target user is a developer or power user who wants a single agent that runs on a $5 VPS or a GPU cluster, hibernates serverlessly when idle, and is reachable from chat platforms while it works on a cloud VM. *(observed fact — README.md, AGENTS.md, pyproject.toml)*

## Layer Map

### Package Inventory

| Package / Module | Role | Public Entrypoints | Key Dependencies | Runtime Surface |
|---|---|---|---|---|
| `agent/` | core semantics | `AIAgent` (via `run_agent.py`), `MemoryProvider` ABC, `ContextEngine`, model adapters (`anthropic_adapter`, `bedrock_adapter`, `auxiliary_client`, `codex_responses_adapter`, `gemini_native_adapter`, `gemini_cloudcode_adapter`, `copilot_acp_client`), `prompt_builder`, `context_compressor`, `curator`, `memory_manager`, `error_classifier`, `redact`, `tool_guardrails`, `credential_pool`, `rate_limit_tracker`, `usage_pricing`, `display.KawaiiSpinner` | `openai`, `anthropic`, `httpx[socks]`, `tenacity`, `rich`, `pydantic`, `jinja2` | in-process Python; called by run_agent / cli / gateway / batch_runner |
| `tools/` | core semantics + integration adapters | `tools.registry.registry` (auto-discovered tool schemas + dispatch), 70+ tool modules (terminal, browser, code-exec, file ops, delegate, mcp, skills, vision, transcription, tts, send_message, session_search, kanban, cron, image gen, RL training, voice, ...) | `httpx`, `firecrawl-py`, `exa-py`, `parallel-web`, `fal-client`, `playwright` (via npm), tool-specific SDKs | imported eagerly by `model_tools.py`; handlers return JSON strings |
| `tools/environments/` | integration adapter | terminal backends: local, docker, singularity, modal, daytona, ssh | `ptyprocess`, `modal` (opt), `daytona` (opt), platform shells | child processes / remote shells launched per agent turn |
| `gateway/` | product shell + protocol layer | `gateway.run` (entrypoint), `gateway.session.GatewaySession`, `gateway.platforms.base.BasePlatformAdapter`, `gateway.platform_registry`, `gateway.stream_consumer`, `gateway.hooks`, `gateway.pairing`, `gateway.delivery`, `gateway.status` | `python-telegram-bot`, `discord.py`, `slack-bolt`, `mautrix`, `aiohttp`, `aiosqlite`, `asyncpg` | long-running async process, multiple platform listeners + per-conversation session lifecycles |
| `gateway/platforms/` | integration adapter | per-platform adapter classes (telegram, discord, slack, whatsapp, signal, matrix, mattermost, email, sms, dingtalk, wecom, weixin, feishu, qqbot, bluebubbles, yuanbao, homeassistant, webhook, api_server) | platform SDKs above + bespoke HTTP clients | each adapter owns its connection lifecycle, scoped credential lock via `acquire_scoped_lock` |
| `hermes_cli/` | product shell | `hermes_cli.main:main` (entry), `commands.COMMAND_REGISTRY`, `setup.py` (wizard), `config.DEFAULT_CONFIG` + `OPTIONAL_ENV_VARS`, `auth.py`, `gateway.py` (subcommand), `models.py`, `model_switch.py`, `tools_config.py`, `skin_engine.py`, `skills_hub.py`, `web_server.py`, `pty_bridge.py`, `doctor.py`, `profiles.py`, `claw.py` (OpenClaw migration), `oneshot.py`, `kanban.py`/`kanban_db.py`, `voice.py`, `webhook.py`, `cron.py`, `relaunch.py` | `prompt_toolkit`, `rich`, `fire`, `simple-term-menu` (legacy), `fastapi`+`uvicorn` (web extra), `ptyprocess` (pty extra) | argparse top-level CLI dispatcher; orchestrates `cli.HermesCLI` interactive REPL (top-level `cli.py`) and Ink TUI (`ui-tui/` + `tui_gateway/`) |
| `cli.py` (top-level) | UI or rendering | `HermesCLI` interactive orchestrator (~11k LOC), `process_command`, `load_cli_config()` | `prompt_toolkit`, `rich`, agent core | interactive REPL; chat surface for `hermes` |
| `tui_gateway/` (top-level) | protocol layer | JSON-RPC backend for Ink TUI; `tui_gateway/server.py` method/event catalog | stdio JSON-RPC newline-delimited | subprocess of Ink Node.js process; long-lived |
| `ui-tui/` (top-level) | UI or rendering | Ink (React) TUI: `entry.tsx`, `app.tsx`, `gatewayClient.ts`, components/hooks/lib | Ink, React, xterm.js (dashboard), TypeScript | Node.js subprocess spawned by `hermes --tui` |
| `acp_adapter/` | protocol layer | `acp_adapter.entry:main` (CLI script `hermes-acp`); ACP server | `agent-client-protocol` (extra) | stdio agent client protocol server (Zed/VS Code/JetBrains) |
| `acp_registry/` | protocol layer | ACP registry support | (unread; inferred from name) | configuration-time registry consumed by `acp_adapter` *(strong inference)* |
| `plugins/` | core semantics + protocol layer | `PluginManager` (in `hermes_cli/plugins.py`), `register(ctx)` API, lifecycle hooks (`pre_tool_call`, `post_tool_call`, `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`); subdirs: `memory/`, `context_engine/`, `image_gen/`, `example-dashboard/`, `disk-cleanup/`, examples | discovered at runtime; entry-point + filesystem | hooks invoked from `model_tools.py` and `run_agent.py`; CLI subcommands wired into argparse |
| `skills/` | core semantics (data) | 30+ category dirs: `apple/`, `autonomous-ai-agents/`, `creative/`, `data-science/`, `devops/`, `github/`, `research/`, `software-development/`, ... | n/a — frontmatter-only `SKILL.md` files | discovered by `agent/skill_commands.py`; injected as user message |
| `optional-skills/` | core semantics (data) | heavier/niche skills installed on demand | n/a | activated via `hermes skills install official/<category>/<skill>` |
| `cron/` | persistence + scheduling | `cron.jobs`, `cron.scheduler` | `croniter` | scheduler thread/loop inside gateway or standalone |
| `environments/` (RL) | integration adapter | RL training environments for Atropos | `atroposlib`, `tinker`, `wandb`, `fastapi`, `uvicorn` (rl extra) | RL trajectory generation surface; out-of-band of normal agent runtime |
| `web/` | UI or rendering | dashboard SPA (Vite/React); served from `hermes_cli/web_dist/` packaged as data | npm-built; runtime served by `hermes_cli/web_server.py` (FastAPI/uvicorn) | localhost SPA + API; binds 127.0.0.1 by default |
| `tests/` | n/a (test) | pytest, marker `integration`, `tests/conftest.py` autouse `_isolate_hermes_home` | `pytest`, `pytest-asyncio`, `pytest-xdist` | local + CI |
| `scripts/` | n/a (build) | `install.sh`, `run_tests.sh`, `release.py` | bash, python | invoked by user / CI |
| `website/` | UI or rendering | Docusaurus docs site | npm | static site; deploys via `deploy-site.yml` |
| Top-level modules (`run_agent.py`, `model_tools.py`, `toolsets.py`, `cli.py`, `hermes_state.py`, `hermes_constants.py`, `hermes_logging.py`, `hermes_time.py`, `batch_runner.py`, `trajectory_compressor.py`, `toolset_distributions.py`, `mcp_serve.py`, `rl_cli.py`, `utils.py`) | core semantics + product shell | `run_agent:main` (CLI script `hermes-agent`), `model_tools.handle_function_call`, `model_tools.discover_builtin_tools`, `mcp_serve` (MCP server entry), `batch_runner.main`, `rl_cli.main`, `hermes_state.SessionDB` (SQLite + FTS5) | declared as py-modules in pyproject | core process modules |

*(All entries above: observed fact — based on AGENTS.md, pyproject.toml, and directory listings of agent/, tools/, gateway/platforms/, hermes_cli/.)*

### Dependency Direction

Lowest stable layer (nothing else depends on it): `tools/registry.py`, `hermes_constants.py`, `hermes_state.py`, `hermes_logging.py`. *(observed fact — AGENTS.md "File Dependency Chain")*

Layer order (low → high):

1. **Bedrock utilities**: `tools/registry.py`, `hermes_constants`, `hermes_state` (SQLite/FTS5 store), `hermes_logging`, `agent/redact`, `agent/file_safety`.
2. **Agent core**: `agent/*` (model adapters, prompt_builder, context_compressor, curator, memory_manager, error_classifier, credential_pool). Imports bedrock; does **not** import tools (tools are passed in as schemas).
3. **Tool implementations**: `tools/*.py` each call `registry.register()` at import time. `model_tools.py` imports `tools/registry` and triggers tool discovery. Tools may import `agent.redact`, `agent.tool_guardrails`. *(observed fact)*
4. **Top-level orchestrators**: `run_agent.py` (`AIAgent` class, ~12k LOC sync loop), `model_tools.py`, `toolsets.py`, `batch_runner.py`. Import agent + tools; export the class consumed by surfaces.
5. **Product shells**: `cli.py` (interactive CLI, ~11k LOC), `hermes_cli/*` (subcommands), `gateway/*` (long-running messaging service), `acp_adapter/*` (IDE adapter), `tui_gateway/*` (Ink RPC backend), `mcp_serve.py` (MCP server).
6. **Plugins / skills / environments**: load on top of all of the above via discovery; invoked via hooks and registry.

**Cycles & wrappers (observed fact / strong inference):**
- `model_tools.py` is a *runtime hub* — it imports tools and is imported by run_agent and gateway. Plugin discovery is a side effect of importing `model_tools.py`; consumers that read plugin state without that import must call `discover_plugins()` explicitly. Documented as a *known pitfall* in AGENTS.md. **(portability hazard — implicit ordering)**
- `_last_resolved_tool_names` is a process-global in `model_tools.py` mutated by `delegate_tool._run_single_child()` save/restore. **(portability hazard — global state)**
- The gateway uses TWO message guards (`gateway/platforms/base.py` `_pending_messages` queue + `gateway/run.py` command interceptor) that must both be bypassed for approval/control commands. **(portability hazard — coupled invariants)**
- `hermes_cli/main.py` (~390KB) is the argparse dispatcher and is allowed to import virtually any package; it sits at the top of the stack and is the only module permitted to wire plugin-specific argparse trees (per the May 2026 "plugins MUST NOT modify core files" rule).

## Public Surfaces

Full catalog in `findings/public-surfaces/public-surfaces.md`. Headline surfaces:

- **CLI binaries (pyproject `[project.scripts]`):** `hermes` → `hermes_cli.main:main`; `hermes-agent` → `run_agent:main`; `hermes-acp` → `acp_adapter.entry:main`. Additional Python entrypoints (declared as py-modules): `mcp_serve`, `batch_runner`, `rl_cli`, `cli`. *(observed fact — pyproject.toml)*
- **`hermes` subcommands (advertised in README):** `hermes`, `hermes model`, `hermes tools`, `hermes config set`, `hermes gateway` (with `setup`/`start`/`run`), `hermes setup`, `hermes claw migrate`, `hermes update`, `hermes doctor`, `hermes logs`, `hermes dashboard`, `hermes whatsapp`, `hermes skills install`, `hermes -p <profile>`, `hermes --tui`. *(observed fact — README, AGENTS.md)*
- **Slash commands (in-conversation, both CLI and gateway):** central `COMMAND_REGISTRY` in `hermes_cli/commands.py` (~65KB). Documented in AGENTS.md: `/new`, `/reset`, `/model`, `/personality`, `/retry`, `/undo`, `/compress`, `/usage`, `/insights`, `/skills`, `/<skill-name>`, `/stop`, `/queue`, `/status`, `/approve`, `/deny`, `/platforms`, `/sethome`, `/help`, `/skin`, `/copy`, `/paste`, `/quit`, `/clear`, `/resume`. *(observed fact — README, AGENTS.md)*
- **Network/RPC surfaces:** Slack Socket Mode, Slack `/hermes` slash subcommands, Telegram BotCommand menu (long-poll + webhook modes, `TELEGRAM_WEBHOOK_URL`), Discord gateway, Matrix client, WhatsApp Baileys bridge, SMS adapter, Email IMAP/SMTP, Webhook receiver (`gateway/platforms/webhook.py`), Home Assistant integration, BlueBubbles, Microsoft Teams (Bot Framework on `TEAMS_PORT` 3978), DingTalk/WeCom/Weixin/Feishu/QQ/Yuanbao adapters; OpenAI-compatible **API server** (`gateway/platforms/api_server.py` ~125KB) gated by `API_SERVER_KEY`+`API_SERVER_HOST`; **Dashboard** (FastAPI `hermes_cli/web_server.py` ~144KB, default 127.0.0.1, `/api/pty` WebSocket, ephemeral `_SESSION_TOKEN`); **MCP server** (`mcp_serve.py`); **ACP server** (`acp_adapter/entry.py`); **TUI JSON-RPC** (`tui_gateway/server.py` over stdio); **RL API server** (`RL_API_URL`, default `http://localhost:8080`). *(observed fact — directory listings, .env.example, docker-compose.yml)*
- **File formats:** `~/.hermes/config.yaml` (settings, versioned `_config_version`), `~/.hermes/.env` (secrets), session JSON trajectories in `logs/session_YYYYMMDD_HHMMSS_UUID.json`, agent.log/errors.log/gateway.log, SQLite session DB (FTS5) via `hermes_state.SessionDB`, kanban DB (`hermes_cli/kanban_db.py` ~105KB), skill `SKILL.md` frontmatter format, plugin entry-point `register(ctx)` ABI, MemoryProvider ABC, MCP-server stdio JSON-RPC. *(observed fact — AGENTS.md, .env.example)*

## Runtime Lifecycle

Full catalog in `findings/runtime-lifecycle/runtime-lifecycle.md`. Summary:

- **Boot — CLI (`hermes`)**: `hermes_cli/main.py` (1) parses `-p <profile>` and calls `_apply_profile_override()` to set `HERMES_HOME` **before any module imports**; (2) loads `.env` via `env_loader.py`; (3) routes to subcommand or to `cli.HermesCLI` interactive REPL; (4) `HermesCLI` calls `load_cli_config()`, builds banner, initializes skin from `display.skin`, scans `~/.hermes/skills/` for slash commands, instantiates `AIAgent`. *(observed fact — AGENTS.md)*
- **Boot — Agent (`run_agent`)**: `AIAgent.__init__` accepts ~60 parameters (credentials, routing, callbacks, session context, budget, credential pool, ...); resolves model via config/provider; loads memory manager + context engine + curator (subject to `skip_memory`); reads context files unless `skip_context_files`; opens prompt-cache-stable system prompt via `prompt_builder`. *(observed fact — AGENTS.md)*
- **Main loop — Agent**: `run_conversation()` runs a **synchronous** while-loop over `client.chat.completions.create(...)`; on `tool_calls`, dispatches each through `model_tools.handle_function_call()` (after pre-tool-call hooks); appends tool result; loops until `max_iterations` (default 90) or `iteration_budget.remaining == 0` (with one-turn grace). Interrupt checks (`self._interrupt_requested`) every iteration. Reasoning content stored in `assistant_msg["reasoning"]`. *(observed fact — AGENTS.md)*
- **Boot — Gateway (`hermes-agent` / `hermes gateway start`)**: `gateway/run.py` (~634KB) reads `~/.hermes/config.yaml` raw via `gateway/config.py`; constructs platform adapters from `platform_registry`; each adapter calls `acquire_scoped_lock()` from `gateway.status`; spawns one async listener per platform; opens stream consumer; per-conversation `GatewaySession` instances are created on demand, each owning an `AIAgent`. *(observed fact — AGENTS.md, gateway dir listing)*
- **Boot — TUI**: `hermes --tui` launches Node.js Ink (`ui-tui/`) which spawns Python `tui_gateway/server.py` over stdio JSON-RPC; gateway emits `gateway.ready` with skin data. Slash commands either run locally (built-ins) or are routed via `slash.exec` to a persistent `_SlashWorker` subprocess and fall through to `command.dispatch`. *(observed fact — AGENTS.md)*
- **Boot — Dashboard**: `hermes dashboard` starts FastAPI/uvicorn from `hermes_cli/web_server.py`; `/api/pty?token=<ephemeral _SESSION_TOKEN>` upgrades to WebSocket and spawns `hermes --tui` over a PTY (`ptyprocess`). Resize via `\x1b[RESIZE:<cols>;<rows>]` and `TIOCSWINSZ`. Localhost-only by default. *(observed fact — AGENTS.md)*
- **Background work**: cron scheduler (`cron/`) runs jobs whose results are delivered to any platform via `gateway.delivery`. Background terminal processes optionally trigger a new agent turn via `display.background_process_notifications` (`all` / `result` / `error` / `off`). *(observed fact — AGENTS.md, .env.example)*
- **Shutdown**: each platform adapter releases its scoped credential lock; gateway awaits in-flight sessions; CLI flushes session DB and trajectory file (if `save_trajectories`). *(strong inference — adapter pattern documented but shutdown sequence not directly read)*

## Concurrency Model

- **Agent core is synchronous** — `run_conversation` is a plain `while` loop with blocking model calls; tool dispatch is sequential (`for tool_call in response.tool_calls`). *(observed fact — AGENTS.md "Agent Loop")* **(portability hazard — synchronous loop wrapped inside async gateway means re-implementations must keep the LLM call blocking-by-design or carefully translate to coroutines)**
- **Gateway is asyncio** — uses `aiohttp`, `python-telegram-bot[webhooks]`, `discord.py`, `slack-bolt`, `mautrix[encryption]`, `aiosqlite`, `asyncpg`. Each platform adapter is an async class; `gateway/platforms/base.py` (~131KB) is the common abstract adapter. *(observed fact — pyproject.toml extras + dir listing)*
- **Per-conversation session lifecycle**: `gateway/session.py` (`GatewaySession`, ~56KB) owns a session per `(platform, channel/user)` pair and synchronizes message queueing. `_pending_messages` in `gateway/platforms/base.py` queues incoming messages while a session is `_active_sessions`. **(portability hazard — interaction between two guards is fragile, per AGENTS.md "two message guards" pitfall)**
- **Credential pooling**: `agent/credential_pool.py` (~67KB) handles round-robin / quota tracking across multiple keys. *(observed fact — agent dir listing + AGENTS.md AIAgent params include `credential_pool`)*
- **Rate limiting / backpressure**: `agent/rate_limit_tracker.py`, `agent/nous_rate_guard.py`, `gateway/platforms/signal_rate_limit.py` (signal-specific). Tenacity for retries. *(observed fact — file listing)*
- **Subagents / parallelism**: `tools/delegate_tool.py` (~107KB) runs isolated child agents via `_run_single_child` which save/restore the `_last_resolved_tool_names` global. **(portability hazard)**
- **Profile token locks**: `gateway.status.acquire_scoped_lock` / `release_scoped_lock` ensure two profiles don't share the same bot credential. *(observed fact — AGENTS.md profile rule 5)*
- **Threading model in CLI/TUI**: `prompt_toolkit` runs its own event loop, `patch_stdout` for non-interfering output; KawaiiSpinner renders from a thread-safe display. The Ink TUI runs in a separate Node.js process. *(observed fact — AGENTS.md, agent/display.py listing)*
- **Process-global state to be aware of**: `_last_resolved_tool_names` (model_tools), `_BUILTIN_SKINS` cache (skin_engine), tool registry (loaded once at first import), discovery-via-import-side-effects (plugins). **(portability hazard cluster)**

## Build and Packaging

Full catalog in `findings/build-and-deploy/build-and-deploy.md`. Summary:

- **Python**: setuptools/`pyproject.toml`, build via `uv venv` + `uv pip install -e .[all,dev]`. Many optional extras: `modal`, `daytona`, `vercel`, `messaging`, `slack`, `matrix`, `cli`, `tts-premium`, `voice`, `pty`, `honcho`, `mcp`, `homeassistant`, `sms`, `acp`, `mistral`, `bedrock`, `web`, `rl`, `yc-bench`, `dingtalk`, `feishu`, `google`, `termux`, `all`. `[all]` excludes `[matrix]` on non-Linux. **`uv` lockfile is used and pinned**, `[tool.uv].exclude-newer = "7 days"`. *(observed fact — pyproject.toml)*
- **Node**: top-level `package.json` (browser tools — npm install + Playwright Chromium), plus `web/` (dashboard SPA, Vite/React build), `ui-tui/` (Ink TUI; sub-package `ui-tui/packages/hermes-ink/`). The Ink TUI is rebuilt by `npm run build` and re-symlinked into `node_modules/@hermes/ink` with a stripped React. *(observed fact — Dockerfile)*
- **Docker**: `Dockerfile` is a multi-stage build (`uv` from `ghcr.io/astral-sh/uv:0.11.6-python3.13-trixie` + `gosu` 1.19 + `debian:13.4`). `tini` reaps zombies; non-root `hermes` UID 10000 with runtime UID/GID remap via gosu in `docker/entrypoint.sh`; Playwright browsers under `/opt/hermes/.playwright`; `HERMES_HOME=/opt/data` mounted as volume. `docker-compose.yml` ships `gateway` + `dashboard` services, both `network_mode: host`, dashboard binding 127.0.0.1 by default. *(observed fact — Dockerfile, docker-compose.yml)*
- **Nix**: `flake.nix` plus `.github/workflows/nix.yml` and `nix-lockfile-fix.yml`. *(observed fact — workflows listing)*
- **Install scripts**: `scripts/install.sh` (curl-piped), `setup-hermes.sh` (clone-and-go: installs `uv`, creates venv, installs `.[all]`, symlinks `~/.local/bin/hermes`). Termux uses curated `[termux]` extra. **Native Windows is not supported** (WSL2 only). *(observed fact — README)*
- **CI/CD (`.github/workflows/`)**: `tests.yml` (pytest -n auto on Ubuntu, plus separate e2e job, both with API keys unset), `docker-publish.yml` (multi-arch amd64+arm64 to Docker Hub `nousresearch/hermes-agent:latest` on main, `:<tag>` on release; gated to `github.repository == 'NousResearch/hermes-agent'`), `nix.yml`, `nix-lockfile-fix.yml`, `deploy-site.yml` (Docusaurus), `docs-site-checks.yml`, `skills-index.yml`, `supply-chain-audit.yml`, `contributor-check.yml`. *(observed fact — workflows directory listing + tests.yml + docker-publish.yml)*
- **Test wrapper**: `scripts/run_tests.sh` enforces hermetic CI parity (unset all `*_API_KEY`/`*_TOKEN`, TZ=UTC, LANG=C.UTF-8, `-n 4` xdist workers). `tests/conftest.py` autouse `_isolate_hermes_home` fixture redirects `HERMES_HOME` to a temp dir. *(observed fact — AGENTS.md "Testing")*

## Porting Priorities

| Component | Priority | Rationale |
|---|---|---|
| `agent/` core (AIAgent loop, prompt_builder, context_compressor) | core | The system is the loop. Without it nothing works. *(observed fact)* |
| `tools/registry.py` + tool dispatch | core | Every tool depends on it; the LLM needs a tool surface. *(observed fact)* |
| Provider adapters (`anthropic_adapter`, `auxiliary_client`, `bedrock_adapter`, `codex_responses_adapter`, `gemini_*_adapter`) | core | Multi-provider support is the value proposition. Each adapter encodes that vendor's tool-call contract. *(strong inference — file sizes 33–167KB indicate vendor-specific quirks)* |
| `tools/terminal_tool.py` + `tools/environments/*` | core | Terminal access is the user-visible workhorse; all 6 backends (local/docker/ssh/modal/daytona/singularity) are advertised. *(observed fact — README)* |
| `tools/file_tools.py`, `tools/file_operations.py`, `tools/code_execution_tool.py` | core | Standard agent file/code tooling; required for parity. *(strong inference)* |
| `agent/memory_manager.py` + `agent/memory_provider.py` + `plugins/memory/*` | core | The "learning loop" differentiator. Honcho/mem0/supermemory/byterover/hindsight/holographic/openviking/retaindb providers. *(observed fact — AGENTS.md)* |
| `gateway/run.py` + `gateway/session.py` + `gateway/platforms/base.py` | important | Multi-platform messaging is the second main value pillar; the base adapter is the contract. *(observed fact)* |
| `gateway/platforms/telegram.py`, `discord.py`, `slack.py` | important | Most-used messaging surfaces; advertised in README. *(observed fact)* |
| `hermes_cli/main.py` + `hermes_cli/setup.py` | important | Setup wizard + dispatcher are the user's entry point; without them onboarding is broken. *(observed fact — file sizes 390KB / 136KB)* |
| `cli.py` (top-level `HermesCLI`) | important | Default chat surface for non-TUI users; ~11k LOC of UX. *(observed fact — AGENTS.md)* |
| `tools/skills_hub.py` + `agent/skill_commands.py` + skills loaders | important | Skills are advertised as a key feature and integrate with the agentskills.io standard. *(observed fact — README)* |
| `tools/delegate_tool.py` | important | Subagent / parallelism — collapses multi-step pipelines into zero-context-cost turns; advertised. *(observed fact — README)* |
| `agent/context_compressor.py` + `agent/curator.py` | important | The pieces that make long sessions viable; closely tied to prompt caching invariant. *(observed fact)* |
| `tools/mcp_tool.py` + `mcp_serve.py` + `tools/mcp_oauth*` | important | MCP integration is heavily advertised and large (~127KB main file). *(observed fact)* |
| `cron/` + `tools/cronjob_tools.py` | important | Built-in scheduler is core to the "runs anywhere" / "unattended" story. *(observed fact)* |
| `hermes_state.py` (SQLite + FTS5 sessions) | important | Session search and resume rely on it; persisted state shape is part of contract. *(observed fact — AGENTS.md)* |
| `acp_adapter/` | optional | IDE integration (Zed/VS Code/JetBrains); useful but not required for first viable port. *(strong inference)* |
| TUI (`ui-tui/` + `tui_gateway/`) | optional | Modern UX layer; the prompt_toolkit `cli.py` already covers the chat surface. Can ship without Ink for MVP. *(strong inference)* |
| Dashboard (`hermes_cli/web_server.py` + `web/`) | optional | Wraps the TUI in a browser; nice-to-have, not core. *(strong inference)* |
| RL training (`environments/`, `tools/rl_training_tool.py`, `rl_cli.py`) | optional | Out-of-band training surface; gated by `[rl]` extra. *(observed fact — pyproject extras)* |
| Optional skills (`optional-skills/`) | optional | Heavy/niche skills installed on demand. *(observed fact)* |
| Chinese-platform adapters (Feishu, DingTalk, WeCom, Weixin, QQ, Yuanbao) | optional | Region-specific; not in `[all]` by default. Very large files (Yuanbao 186KB, Feishu 198KB). *(observed fact)* |
| Skin engine (`hermes_cli/skin_engine.py`) | incidental | Visual customization only; no behavioral impact. *(observed fact — AGENTS.md)* |
| KawaiiSpinner / branding text | incidental | Cosmetic. *(observed fact)* |
| Honcho-specific argparse glue / OpenClaw migration (`hermes claw migrate`, `hermes_cli/claw.py`) | incidental | One-time migration helper. *(observed fact — README)* |

## Durable State

Full catalog in `findings/state-and-storage/state-and-storage.md`. Headlines:

- **Config**: `~/.hermes/config.yaml` (settings, versioned `_config_version`), `~/.hermes/.env` (secrets only — API keys, tokens, passwords). Profile-aware via `HERMES_HOME`; per-profile dir at `~/.hermes/profiles/<name>/`. **`MESSAGING_CWD` and `TERMINAL_CWD` env vars are deprecated** — canonical setting is `terminal.cwd` in config.yaml. *(observed fact — AGENTS.md)*
- **Sessions**: SQLite session store with FTS5 search via `hermes_state.SessionDB`; trajectory JSON files in `logs/session_YYYYMMDD_HHMMSS_UUID.json`. *(observed fact — AGENTS.md, .env.example)*
- **Logs**: `~/.hermes/logs/agent.log` (INFO+), `errors.log` (WARNING+), `gateway.log` (when gateway is running). Browse via `hermes logs [--follow] [--level ...] [--session ...]`. *(observed fact — AGENTS.md)*
- **Skills**: `~/.hermes/skills/` (user-created) + repo `skills/` (built-in) + `optional-skills/` (heavier). OpenClaw imports go to `~/.hermes/skills/openclaw-imports/`. *(observed fact — README)*
- **Plugins**: `~/.hermes/plugins/`, `./.hermes/plugins/`, and pip entry points. *(observed fact — AGENTS.md)*
- **Skins**: `~/.hermes/skins/*.yaml` (user-installed). *(observed fact — AGENTS.md)*
- **Auth/secrets**: `auth.py` (~183KB) manages tokens; per-platform credentials (Telegram, Slack, Discord, Matrix, etc.); `tools/credential_files.py`; Qwen reuses `~/.qwen/oauth_creds.json`; Honcho uses `~/.honcho/config.json`. *(observed fact — file listing, .env.example)*
- **Caches**: skin cache, model catalog cache (`hermes_cli/model_catalog.py`), Playwright browsers in container (`/opt/hermes/.playwright`), browser session state, sticker cache (`gateway/sticker_cache.py`).
- **Kanban DB**: `hermes_cli/kanban_db.py` (~105KB) — SQLite-backed. *(observed fact — file listing)*
- **Generated artifacts**: `hermes_cli/web_dist/**` (built dashboard SPA), `node_modules/@hermes/ink` (built Ink TUI), trajectory files, RL training outputs (wandb), session DB.

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| arch-OQ1 | open question | Exact shutdown sequence and ordering of locks (credential pool, scoped platform locks, session flush) — confirmed by AGENTS.md that locks exist, not by reading the shutdown code path. | Would require reading `gateway/run.py` (~634KB) and `gateway/platforms/base.py` (~131KB); deferred to keep within 30-file read budget. |
| arch-OQ2 | open question | Whether `acp_registry/` is purely configuration or has runtime behavior; not read. | Directory existed in pre-loaded recon but no module within it was read. |
| arch-OQ3 | open question | Whether `cli.py` (top-level legacy CLI, ~11k LOC) is wholly replaced by `hermes_cli/` or still has unique features. | AGENTS.md describes both as live; precise overlap not verified. |
| arch-OQ4 | open question | Exact concurrency primitives used inside `gateway/session.py` for the two-guard message queue (asyncio.Lock vs custom semaphore). | Not read directly; AGENTS.md describes the contract but not the primitive. |
| arch-OQ5 | open question | Whether the OpenAI-compatible `gateway/platforms/api_server.py` reuses `AIAgent` directly or wraps it in a session-pool. | File is ~125KB; not read. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| arch-CF1 | contracts | Full slash-command catalog with trigger, defaults, side effects, persisted state, error behavior — only headline commands enumerated here. | Contracts phase rubric covers per-feature trigger/defaults/outputs; would be premature here. |
| arch-CF2 | protocols | Wire-format / JSON-RPC method catalog for `tui_gateway/server.py` (Ink-Python), MCP server schema, ACP server schema, and OpenAI-compatible API server schema. | Protocols phase explicitly covers event catalogs and persistence formats. |
| arch-CF3 | protocols | Persistent schema details: `hermes_state` SQLite/FTS5 schema, `kanban_db` schema, session-trajectory JSON shape, config.yaml `_config_version` migration table. | Persistence schema notes belong in protocols. |
| arch-CF4 | protocols | Gateway message-queue state machine (the "two guards" pattern: `_pending_messages` + command interceptor) and per-platform session lifecycle. | State-machine extraction is the protocols phase's job. |
| arch-CF5 | defect-scan-mechanical | `_last_resolved_tool_names` process-global mutated by `delegate_tool._run_single_child`; AGENTS.md flags it as a known pitfall. | Concrete bug surface, not architectural. |
| arch-CF6 | defect-scan-mechanical | Plugin discovery is a side-effect of importing `model_tools.py`; consumers that read plugin state without that import must call `discover_plugins()` explicitly. | Mechanical defect surface (initialization order). |
| arch-CF7 | defect-scan-mechanical | ANSI `\033[K` leaks as literal text under `prompt_toolkit.patch_stdout` per AGENTS.md known-pitfall list. | Mechanical defect to verify by grep. |
| arch-CF8 | defect-scan-semantic | Two-message-guard interaction (`_pending_messages` + gateway/run.py command interceptor) — "any new command that must reach the runner while the agent is blocked must bypass BOTH guards." Concurrency contract violation. | Concurrency reasoning belongs to the deep semantic pass. |
| arch-CF9 | defect-scan-semantic | Prompt-cache invariant: "do not alter past context, change toolsets, or rebuild system prompts mid-conversation" — easy to violate via slash-command implementations. | Semantic invariant tied to LLM cost behavior. |
| arch-CF10 | porting | Porting hazard: `auxiliary_client.py` is 167KB; consolidating multi-provider auxiliary calls cleanly is non-trivial. Capture target-language design before writing. | Porting phase rubric. |
| arch-CF11 | reimplementation-spec | Decision: which of the 18+ messaging platforms make the MVP cut for a re-implementation, and which become later-phase ports. | Decision rests on the reimplementation-spec strategic alignment hook. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system intent is documented. | PASS | §System Intent (one paragraph covering what, who-for, problem-solved, differentiators). |
| 2 | The layer map and dependency direction are documented. | PASS | §Layer Map → Package Inventory table (covers all required packages: agent, tools, gateway, hermes_cli, skills, plugins, cron, acp_adapter, acp_registry, web, environments, plus tools/environments, ui-tui, tui_gateway, top-level modules). §Layer Map → Dependency Direction lists 6 layers low→high and explicitly calls out cycles (model_tools as runtime hub, `_last_resolved_tool_names` global, two-guard pattern, hermes_cli/main.py wrapper status). |
| 3 | Public surfaces are identified. | PASS | §Public Surfaces enumerates CLI binaries, hermes subcommands, slash commands, network/RPC interfaces, file formats, with headline list in primary and full enumeration in `findings/public-surfaces/public-surfaces.md`. |
| 4 | Runtime lifecycle, concurrency model, and porting priorities are summarized. | PASS | §Runtime Lifecycle (boot per surface, agent loop body, gateway loop, shutdown). §Concurrency Model (sync agent / async gateway split, two-guard pattern, credential pool, profile token locks, process-global hazards). §Porting Priorities table has 25+ rows across `core`/`important`/`optional`/`incidental` (above the 12-row floor). |
| 5 | Findings are marked with evidence levels. | PASS | All material claims annotated with `observed fact`, `strong inference`, `portability hazard`, or `open question`. Open questions also recorded in §Open Questions table; deferrals tracked in §Carry-Forward table with explicit `target_phase`. |

**Validated by:** 2026-05-06 (architecture phase, codecarto session for hermes-agent)
**Overall:** PASS

Notes on completeness: 5 entries in §Open Questions reflect honest gaps tied to the 30-file read budget (notably arch-OQ1 shutdown ordering, arch-OQ4 session-queue concurrency primitives, arch-OQ5 api_server agent reuse). 11 entries in §Carry-Forward route deferred work to the appropriate downstream phase. No criterion is FAIL or PARTIAL given the rubric's wording — depth gaps belong in the open-questions / carry-forward lists per the validation guide's "PARTIAL only when the criterion itself is unmet" rule.
