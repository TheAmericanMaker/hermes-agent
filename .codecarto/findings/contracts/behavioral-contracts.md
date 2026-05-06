# Behavioral Contracts

<!-- Phase 3 (contracts) primary output for hermes-agent. Codecarto session 2026-05-06. -->

## Surfaces Covered

Hermes Agent exposes its tool-calling agent loop through six categories of user-facing surface. Per-surface ownership maps back to the layers in `findings/architecture/architecture-map.md`. *(observed fact — pyproject.toml `[project.scripts]`, AGENTS.md "Project Structure", docker-compose.yml)*

- **CLI (interactive REPL).** `hermes` console script → `hermes_cli.main:main` dispatches subcommands or invokes the legacy top-level `cli.HermesCLI` (~11k LOC, prompt_toolkit + rich) for the chat REPL. Owner: `hermes_cli/` + top-level `cli.py`.
- **CLI (non-interactive subcommands).** `hermes setup`, `hermes config set`, `hermes model`, `hermes tools`, `hermes gateway {setup,start,run}`, `hermes claw migrate`, `hermes update`, `hermes doctor`, `hermes logs`, `hermes dashboard`, `hermes whatsapp`, `hermes skills install`, `hermes voice`, `hermes webhook`, `hermes cron`, `hermes kanban`, `hermes uninstall`, plus `hermes -p <profile>` for multi-instance isolation. Owner: `hermes_cli/`.
- **TUI.** `hermes --tui` (or env `HERMES_TUI=1`) spawns Node.js Ink (`ui-tui/`) which talks newline-delimited JSON-RPC over stdio to a Python `tui_gateway/server.py` subprocess. Owner: `ui-tui/` (TS) + `tui_gateway/` (Py).
- **Web UI.** `hermes dashboard` serves a Vite/React SPA (`web/`) via FastAPI/uvicorn (`hermes_cli/web_server.py`). The chat page (`/chat`) embeds `hermes --tui` over a PTY WebSocket (`/api/pty?token=…`). Default bind `127.0.0.1`. Owner: `hermes_cli/web_server.py` + `web/`.
- **API/SDK.** Three RPC servers: (a) MCP server `mcp_serve.py` (stdio JSON-RPC, `[mcp]` extra), (b) ACP server `acp_adapter/entry.py` for IDE clients (Zed/VS Code/JetBrains, `[acp]` extra), (c) OpenAI-compatible HTTP API `gateway/platforms/api_server.py` (gated by `API_SERVER_KEY`+`API_SERVER_HOST`). Owner: respective modules.
- **Bot/gateway (multi-platform messaging).** `hermes-agent` / `hermes gateway start` runs `gateway/run.py`, fan-out to per-platform adapters in `gateway/platforms/`: Telegram, Discord, Slack, WhatsApp (Baileys), Signal, Matrix, Email (IMAP+SMTP), SMS, Mattermost, Microsoft Teams (Bot Framework, port 3978), Home Assistant, BlueBubbles, Webhook, plus Chinese platforms DingTalk/WeCom/Weixin/Feishu/QQ/Yuanbao. Each adapter holds a scoped credential lock via `gateway.status.acquire_scoped_lock`. Owner: `gateway/`.
- **Storage / export formats.** SQLite session DB with FTS5 (`hermes_state.SessionDB`), JSON trajectory files in `logs/session_YYYYMMDD_HHMMSS_UUID.json`, kanban SQLite (`hermes_cli/kanban_db.py`), config YAML (`~/.hermes/config.yaml`, versioned `_config_version`), secrets `.env`, skills tree (`~/.hermes/skills/<cat>/<skill>/SKILL.md` with YAML frontmatter), skin YAML (`~/.hermes/skins/*.yaml`), plugins tree, MCP OAuth credential cache. Owner: `hermes_state.py`, `hermes_cli/`, plus per-feature stores. *(observed fact — AGENTS.md, .env.example)*

## Feature Contracts

Grouped by surface. Each row: trigger / defaults / observable output / side effects / persisted state / error behavior / retry-recovery / owner. Evidence labels: `OF` = observed fact, `SI` = strong inference, `PH` = portability hazard.

### CLI / TUI / Gateway — slash commands (in-conversation)

| # | Feature | Trigger / input | Defaults | Observable output | Side effects | Persisted state | Error behavior | Retry / recovery | Owner |
|---|---|---|---|---|---|---|---|---|---|
| C1 | `/new`, `/reset`, `/clear` (new session) | User types in CLI/TUI/gateway | n/a | New session ID, transcript cleared, banner re-printed | Closes prior `AIAgent`, opens new one; new row in `SessionDB`; new trajectory file (if `save_trajectories`) | Session DB row, trajectory JSON | If session flush fails, prior session may be partially-flushed | New session always starts; user re-issues prompt | `cli.HermesCLI`, `gateway/run.py` (OF — AGENTS.md slash registry) |
| C2 | `/model [provider:model]` | User input; arg optional | Falls back to `model.default` from config.yaml | Confirms switch; next turn uses new model | Updates **session-local** model; does NOT alter system prompt mid-conversation (cache invariant) | None (session scope) | Unknown model → error message, model unchanged | User re-issues with valid id | `hermes_cli/model_switch.py`, `cli.py` (OF — commands.py registry; AGENTS.md cache rule) |
| C3 | `/personality [name]` | User input; arg optional | List on no-arg | New persona injected next turn | Same cache-aware injection rule (PH — must not rebuild system prompt mid-conv) | Active persona name in config | Unknown persona → list shown | User retries | `cli.HermesCLI` (OF) |
| C4 | `/retry` | User types | n/a | Last user message resent; new assistant response | Replaces last assistant msg in history; new trajectory entry | SessionDB updated | If model errors, reports error and leaves prior reply intact | User can `/retry` again or `/undo` | `cli.HermesCLI` (OF) |
| C5 | `/undo` | User types | n/a | Last (user, assistant) exchange removed | Mutates history list; trajectory file gets undo marker | SessionDB updated | Cannot undo at session start | n/a | `cli.HermesCLI` (OF) |
| C6 | `/compress` | Manual; auto-trigger when context >= `compression.threshold` (default 0.85) | Summary model `google/gemini-3-flash-preview` | Middle turns replaced with summary | LLM call to summary model; truncates messages list | Compression marker in trajectory | If summary model fails, abort compression and keep raw history (SI) | Auto-retry next turn | `agent/context_compressor.py` (OF — .env.example, AGENTS.md "Context Compression") |
| C7 | `/usage`, `/insights [--days N]` | User types | `--days 30` for insights | Token usage + rate-limit panel | Reads from rate_limit_tracker, usage_pricing | None (read-only) | Missing data → "no data" message | n/a | `agent/rate_limit_tracker.py`, `agent/usage_pricing.py` (OF) |
| C8 | `/skills`, `/<skill-name>`, `/skills install [--now]` | User types | `--now` opt-in for immediate cache invalidation | Browse / activate skill; on activation skill content injected as **user message** (not system prompt — preserves prompt cache) | `~/.hermes/skills/` scanned; for `install`, files written | Skill tree on disk | Skill not found → error; install network failure → reported | User re-runs install | `agent/skill_commands.py`, `tools/skills_hub.py` (OF — AGENTS.md, cache-aware policy) |
| C9 | `/stop` (gateway) / interrupt (CLI Ctrl-C) | Slash in messaging or SIGINT in CLI | n/a | Running agent halts ASAP at next iteration boundary | Sets `_interrupt_requested`; bypasses BOTH gateway message guards (`_pending_messages` + command interceptor) | Partial trajectory saved (SI) | Tool call already in flight may finish before interrupt is observed | User re-prompts | `run_agent.AIAgent`, `gateway/run.py` (OF — AGENTS.md "two message guards" pitfall; PH) |
| C10 | `/queue <prompt>`, `/status` | User types in gateway | n/a | Prompt queued for next turn / queue inspected | `_pending_messages` queue mutated | None | n/a | n/a | `gateway/platforms/base.py`, `gateway/run.py` (OF — commands.py) |
| C11 | `/approve`, `/deny` | User responds to dangerous-command prompt | n/a (must follow approval request) | Tool either executes or returns deny-result to model | Bypasses both message guards inline; consumes pending approval token | Approval logged in trajectory | If no pending approval -> "nothing to approve" | User issues new turn | `tools/tool_guardrails`, `gateway/run.py` (OF — AGENTS.md) |
| C12 | `/sethome` (gateway) | User in messaging client | n/a | Sets current chat as home channel for cron delivery, etc. | Writes channel id to per-platform config (e.g. `TELEGRAM_HOME_CHANNEL`) | `~/.hermes/config.yaml` or .env | If write fails -> error message | User retries | `gateway/run.py`, platform adapter (OF) |
| C13 | `/skin [name]` | User types | List on no-arg; deferred invalidation by default; `--now` for immediate | Skin colors/spinner/branding swap | `_BUILTIN_SKINS` cache lookup or YAML load from `~/.hermes/skins/` | `display.skin` saved to config.yaml | Unknown skin -> fallback to `default`, warning shown | User picks valid name | `hermes_cli/skin_engine.py` (OF — AGENTS.md "Skin/Theme System") |
| C14 | `/copy`, `/paste`, `/image <path>` | User types | n/a | Last response copied to clipboard / clipboard image attached / file attached as next-turn input | OS clipboard read/write; image bytes uploaded with next message | None (transient) | Clipboard unavailable -> error | User retries | `cli.HermesCLI` (OF — commands.py) |
| C15 | `/quit`, `/resume` | User types | `/resume` lists sessions | CLI exits / prior session restored | `/resume` reads SessionDB and re-hydrates `AIAgent` | SessionDB read | Resume on missing id -> list shown | User picks valid id | `cli.HermesCLI`, `hermes_state.SessionDB` (OF) |
| C16 | `/yolo` | User types | OFF (deny by default) | Toggles bypass of dangerous-command approvals for this session | Session-scoped flag | None | n/a (toggle) | `/yolo` again to disable | `cli.HermesCLI`, `tools/tool_guardrails.py` (OF — registry "skip all dangerous command approvals") |
| C17 | `/cron`, `/kanban`, `/curator` | User types | Subcommand-driven | List/add/delete jobs / kanban TUI / background-curator status | Cron job DB, kanban SQLite, curator state | `cron/` DB, kanban DB | Subcommand parse error -> usage shown | User retries | `cron/`, `hermes_cli/kanban_db.py`, `agent/curator.py` (OF) |
| C18 | `/tools`, `/toolsets`, `/plugins` | User types | List view | Tool/toolset/plugin status table | Reads `tools.registry`, `PluginManager` | None (read-only) | Plugin import error -> degraded list with warning | User reloads | `tools/registry.py`, `hermes_cli/plugins.py` (OF) |

### CLI subcommands (shell-level, not slash)

| # | Feature | Trigger | Defaults | Observable output | Side effects | Persisted state | Error behavior | Owner |
|---|---|---|---|---|---|---|---|---|
| S1 | `hermes setup` | User shell | First-run wizard prompts for provider/model/keys | Writes config.yaml + .env; OpenClaw migration prompt | Creates `~/.hermes/`, writes both files | YAML+ENV | Permission errors fail loudly | `hermes_cli/setup.py` (OF — AGENTS.md) |
| S2 | `hermes -p <profile>` | Process invocation | Default profile if unset | Sets `HERMES_HOME=~/.hermes/profiles/<name>` BEFORE module imports | All subsequent `get_hermes_home()` reads scoped | Per-profile dir | Profile not found -> created on first use | `hermes_cli/profiles.py`, `hermes_cli/main.py` `_apply_profile_override()` (OF — AGENTS.md "Profiles") |
| S3 | `hermes gateway start` / `hermes-agent` | User shell | Reads config.yaml; spawns one async listener per enabled platform | Long-running process; logs to gateway.log | Per-platform connections; scoped credential locks acquired | `gateway.status` lock files | Platform connect failure -> that platform logs error and is skipped; others continue (SI) | `gateway/run.py` (OF) |
| S4 | `hermes dashboard` | User shell | Bind 127.0.0.1; ephemeral `_SESSION_TOKEN` | Browser SPA + PTY WebSocket | Spawns `hermes --tui` per WS session | Session token in memory only | Token mismatch -> 401; PTY death -> WS close (SI — PH for native Windows: WSL2-only) | `hermes_cli/web_server.py`, `pty_bridge.py` (OF) |
| S5 | `hermes claw migrate` | User shell | `--dry-run` recommended; `--preset`/`--overwrite` modifiers | Imports OpenClaw skills/config | Writes to `~/.hermes/skills/openclaw-imports/` | New skill tree entries | Conflict -> preserved unless `--overwrite` | `hermes_cli/claw.py` (OF — README) |
| S6 | `hermes logs [--follow] [--level X] [--session ID]` | User shell | Level INFO+ | Tails `~/.hermes/logs/agent.log` etc. | None | Reads log files | File missing -> empty output | `hermes_cli/logs.py` (OF) |
| S7 | `hermes config set <dotpath> <value>` | User shell | n/a | Persists to `~/.hermes/config.yaml` via deep-merge | YAML write | config.yaml | Type mismatch -> validation error | `hermes_cli/config.py` (OF — AGENTS.md "Adding Configuration") |

### API / SDK / Background

| # | Feature | Trigger | Defaults | Observable output | Side effects | Persisted state | Error behavior | Owner |
|---|---|---|---|---|---|---|---|---|
| A1 | OpenAI-compatible HTTP API | POST to `gateway/platforms/api_server.py` endpoint, header `Authorization: Bearer $API_SERVER_KEY` | Off unless `API_SERVER_KEY`+`API_SERVER_HOST` set | OpenAI-shape response (chat completions) | New `AIAgent`/session per request (SI — open question arch-OQ5) | SessionDB row | Bad/missing key -> 401; rate-limit -> 429 (SI) | `gateway/platforms/api_server.py` (OF — recon) |
| A2 | MCP server (stdio JSON-RPC) | `mcp_serve` invocation by client | `[mcp]` extra | Tool/resource discovery + invocation per MCP spec | Imports `tools.registry`, may invoke real tools | Tool side-effects only | Spec violations -> JSON-RPC error | `mcp_serve.py` (OF) |
| A3 | ACP server (stdio JSON-RPC) | `hermes-acp` invoked by IDE | `[acp]` extra | ACP message frames | Spawns/manages an `AIAgent` per session | Session DB row | Disconnect -> graceful exit | `acp_adapter/entry.py` (OF) |
| A4 | Cron scheduler | `cron/scheduler` polling tick | n/a | Scheduled prompt submitted to gateway -> results delivered to home channel via `gateway.delivery` | New session per job | Cron DB; SessionDB | Job failure logged; retry per job config | `cron/jobs.py`, `cron/scheduler.py` (OF) |
| A5 | Background terminal-process watcher | `terminal(background=true, notify_on_complete=true)` | `display.background_process_notifications=all` | New agent turn triggered on completion | Watcher thread per bg process | Trajectory entries | Watcher death silently drops notification (SI — open question) | `tools/terminal_tool.py`, gateway watcher (OF — AGENTS.md) |
| A6 | Tool dispatch | LLM emits `tool_calls` in chat response | Sequential per response | Tool result appended to message history | Tool's own side-effects (filesystem, network, terminal) | Per-tool | Handler must return JSON string; errors wrapped by registry | `model_tools.handle_function_call`, `tools/registry.py` (OF — AGENTS.md "Adding New Tools") |
| A7 | Subagent / delegate | Parent agent calls `delegate_tool` | Inherits parent toolsets unless overridden | Child's final response returned as tool result | `_run_single_child` saves/restores `_last_resolved_tool_names` global (PH) | Child trajectory file | Child error bubbles up as tool error | `tools/delegate_tool.py` (OF — AGENTS.md pitfalls) |

### Storage / Export

| # | Feature | Trigger | Defaults | Observable output | Side effects | Persisted state | Error behavior | Owner |
|---|---|---|---|---|---|---|---|---|
| ST1 | Session DB (SQLite + FTS5) | Every turn | Per-profile path | n/a (queryable) | Insert/update rows; FTS index updated | `hermes_state.db` | Lock contention -> SQLite retry (SI) | `hermes_state.SessionDB` (OF) |
| ST2 | Session trajectory JSON | Per session if `save_trajectories=true` | Off by default | `logs/session_YYYYMMDD_HHMMSS_UUID.json` | Append-only file write | Disk file | Disk full -> exception logged, session continues (SI) | `agent/trajectory.py` (OF — .env.example) |

## High-Value Behaviors

- **Cancellation / abort.** Synchronous agent loop checks `self._interrupt_requested` once per iteration and at budget boundaries; no hard-kill mid-tool-call. In gateway, `/stop` and friends MUST bypass two message guards (`_pending_messages` queue in `gateway/platforms/base.py` AND command interceptor in `gateway/run.py`) — otherwise they get queued behind the running agent. *(observed fact — AGENTS.md "two message guards" pitfall) (portability hazard)*
- **Streaming / partial output.** TUI streams via `message.delta`/`message.complete` JSON-RPC events. CLI uses `KawaiiSpinner` + activity feed during blocking calls. Gateway honors `HERMES_HUMAN_DELAY_MODE` (`off`/`natural`/`custom`) for human-pacing. *(OF — AGENTS.md TUI table, .env.example)*
- **Queueing / follow-up.** `/queue <prompt>` adds to `_pending_messages` for next turn without interrupting current. `/steer` injects a message between tool calls without forcing a new turn. Gateway also queues messages received while `session_key in _active_sessions`. *(OF — commands.py registry)*
- **Compaction / summarization.** Auto when `compression.enabled=true` and ratio >= `compression.threshold` (default 0.85); summarized via `compression.summary_model` (default `google/gemini-3-flash-preview`). Manual via `/compress`. Compression is the **only** allowed mid-conversation context mutation — all other mutations defer to next session by policy. *(OF — AGENTS.md "Prompt Caching Must Not Break")*
- **Persistence / resume.** Every session writes to SQLite + FTS5 (`hermes_state.SessionDB`) and optionally a trajectory JSON. `/resume` lists and re-hydrates a prior session. Profiles isolate state under `~/.hermes/profiles/<name>/`. *(OF — AGENTS.md)*
- **Tool execution & validation.** Registry collects schemas at import time; LLM-emitted tool_calls are routed through `model_tools.handle_function_call`; dangerous commands gated by `tools/tool_guardrails.py` (approval prompt -> `/approve`/`/deny`); plugin pre/post hooks invoked. All handlers must return JSON strings; the registry wraps exceptions. `/yolo` bypasses dangerous-command approvals for the session. *(OF — AGENTS.md "Adding New Tools")*

## Security and Authorization

**Authentication.**
- LLM providers: per-vendor API keys in `~/.hermes/.env` (OPENROUTER_API_KEY, GOOGLE_API_KEY, OLLAMA_API_KEY, GLM_API_KEY, KIMI_API_KEY, ARCEEAI_API_KEY, MINIMAX_API_KEY, OPENCODE_ZEN_API_KEY, OPENCODE_GO_API_KEY, HF_TOKEN, XIAOMI_API_KEY, GROQ_API_KEY, VOICE_TOOLS_OPENAI_KEY); Qwen via OAuth (reuses `~/.qwen/oauth_creds.json`). PyJWT used for token decoding.
- Gateway platforms: per-platform OAuth tokens / bot tokens (TELEGRAM_BOT_TOKEN, SLACK_BOT_TOKEN+SLACK_APP_TOKEN, EMAIL_PASSWORD app-password, TEAMS_CLIENT_ID/SECRET/TENANT, WHATSAPP via Baileys pairing).
- Tools: BROWSERBASE_API_KEY+PROJECT_ID, EXA_API_KEY, PARALLEL_API_KEY, FIRECRAWL_API_KEY, FAL_KEY, HONCHO_API_KEY, GITHUB_TOKEN, TINKER_API_KEY, WANDB_API_KEY.
- HTTP API server: shared-secret `API_SERVER_KEY` (Bearer header).
- Dashboard: ephemeral `_SESSION_TOKEN` passed as query param on `/api/pty?token=...` (browsers cannot set Authorization header on WS upgrade). *(OF — AGENTS.md TUI dashboard section)*
- MCP OAuth: per-server credentials managed by `tools/mcp_oauth.py` + `mcp_oauth_manager.py`.

**Authorization model.**
- Per-platform allowlists: `TELEGRAM_ALLOWED_USERS`, `SLACK_ALLOWED_USERS`, `WHATSAPP_ALLOWED_USERS`, `EMAIL_ALLOWED_USERS`, `TEAMS_ALLOWED_USERS` (comma-separated IDs).
- Global override `GATEWAY_ALLOW_ALL_USERS=true` and `TEAMS_ALLOW_ALL_USERS=true` skip allowlists (default `false` = deny). *(observed fact — .env.example, portability hazard — single env-var trapdoor for "open access")*
- Tool-level guardrails for dangerous commands require `/approve` (or `/yolo` session-toggle to skip approvals).
- Profile credential isolation: `gateway.status.acquire_scoped_lock` ensures two profiles don't reuse the same bot token. *(OF — AGENTS.md profile rule 5)*

**Trust boundaries.**
- LLM responses are untrusted input — tool schemas constrain what tools the model can invoke; dangerous-command guardrails add a human-approval gate.
- Inbound messages on every gateway platform are untrusted — allowlists and per-platform sender verification (e.g. Telegram bot tokens, Slack signature) enforce.
- `agent/redact.py` redacts sensitive output; `agent/file_safety.py` guards filesystem operations.
- Subagents (`delegate_tool`) inherit/restrict toolsets; child's actions still subject to guardrails.

**Secret management.**
- `.env` is for **secrets only** (API keys / tokens / passwords). Non-secrets live in `config.yaml`. `python-dotenv` loads via `hermes_cli/env_loader.py`.
- SUDO_PASSWORD is plaintext-on-disk — `.env.example` warns "only on trusted machines." *(observed fact — portability hazard)*
- `OPTIONAL_ENV_VARS` in `hermes_cli/config.py` carries metadata (description, prompt, url, password=bool, category) for setup-wizard flow.

**Session lifecycle.**
- Per-conversation `GatewaySession` in gateway — created on first message per (platform, channel/user), torn down via shutdown sequence (open question arch-OQ1 — exact ordering not directly observed).
- CLI session ends on `/quit`; `/resume` re-hydrates from SessionDB.
- Dashboard PTY session ends when `hermes --tui` child exits OR WebSocket closes.

**Web security.**
- Dashboard binds `127.0.0.1` by default. *(OF — docker-compose.yml; AGENTS.md TUI section)*
- No CORS / CSP details directly observed in the read budget — *open question*.

## Configuration Model

Detailed in `findings/config-model/config-model.md`. Summary:

- **Sources, low -> high precedence:** `DEFAULT_CONFIG` (in `hermes_cli/config.py`, versioned `_config_version`) -> `~/.hermes/config.yaml` (deep-merged) -> `~/.hermes/.env` (secrets only) -> process env -> CLI flags -> plugin overrides.
- **Three loaders (PH).** `load_cli_config()` (top-level `cli.py`), `load_config()` (`hermes_cli/config.py`), and direct YAML in `gateway/run.py`+`gateway/config.py`. New keys must be in `DEFAULT_CONFIG` to be visible to all three. *(OF — AGENTS.md "Config loaders (three paths)")*
- **File format.** YAML for config, dotenv for secrets, YAML frontmatter inside `SKILL.md`, JSON for trajectories, SQLite for sessions+kanban.
- **Discovery.** `~/.hermes/` (or `~/.hermes/profiles/<name>/` if `-p <profile>`) anchored on `get_hermes_home()`; profile resolution must happen before any module that caches paths is imported (`_apply_profile_override()`).
- **Feature flags.** Notable: `compression.enabled` / `compression.threshold` / `compression.summary_model`; `display.skin`; `display.background_process_notifications` (`all`/`result`/`error`/`off`); `display.tool_progress_command` (gates a CLI-only command for gateway use via `gateway_config_gate`); `terminal.backend` (one of `local`/`docker`/`singularity`/`modal`/`ssh`/`daytona`); `memory.provider`; `stt.provider`; `BROWSERBASE_PROXIES`/`BROWSERBASE_ADVANCED_STEALTH`; `GATEWAY_ALLOW_ALL_USERS`/`TEAMS_ALLOW_ALL_USERS`; `HERMES_HUMAN_DELAY_MODE`+`MIN_MS`/`MAX_MS`; `HERMES_BACKGROUND_NOTIFICATIONS`. *(OF — AGENTS.md, .env.example)*
- **Cache-aware mutation rule.** Slash commands that change system-prompt state default to **deferred invalidation**; opt-in `--now` flag for immediate. Canonical pattern `/skills install --now`. Violating this breaks prompt caching -> big cost spike. *(OF — AGENTS.md "Important Policies")*
- **Validation.** Deep-merge handles additive config changes automatically; `_config_version` is bumped only when active migration needed. Type/value errors during set surface in `hermes config set` output. Specific schema validation library not observed in budget — *strong inference: no centralized JSON-schema validator beyond pydantic models in tool schemas*.
- **Deprecations enforced.** `MESSAGING_CWD` env removed (warning if set in `.env`); `TERMINAL_CWD` env deprecated, canonical is `terminal.cwd` in config.yaml; `LLM_MODEL` env no longer read.

## Doc/Test Conflicts

None directly observed in the read budget; `tests/conftest.py` autouse `_isolate_hermes_home` enforces the docs' rule "tests must not write to `~/.hermes/`". *(open question — full test-vs-docs sweep deferred to defect-scan-semantic since change-detector tests are explicitly disallowed by AGENTS.md "Don't write change-detector tests")*

## Black-Box Acceptance List

| # | Scenario | Precondition | Action | Expected Outcome |
|---|----------|--------------|--------|------------------|
| 1 | First-run onboarding | Empty `~/.hermes/`, OPENROUTER_API_KEY available | Run `hermes setup`, accept defaults | `~/.hermes/config.yaml` and `~/.hermes/.env` exist; `_config_version` = current; `model.default` populated; `terminal.backend=local`. |
| 2 | Profile isolation | Two profiles `coder` and `ops` configured | `hermes -p coder` then `hermes -p ops` in separate shells | Each shell sees its own config/sessions/skills under `~/.hermes/profiles/<name>/`; bot tokens locked per-profile (no double-connect). *(cite `tests/hermes_cli/test_profiles.py` profile_env fixture)* |
| 3 | Slash `/model` switches without breaking cache | Active CLI session | `/model openrouter:anthropic/claude-sonnet-4` | Confirmation message; next turn uses new model; system prompt unchanged (verified by no cost spike on next short turn). |
| 4 | Auto-compression triggers at 0.85 ratio | Conversation length >=85% of context limit, `compression.enabled=true` | Send next prompt | Middle messages replaced by summary; conversation continues; trajectory JSON shows compression marker. |
| 5 | `/stop` interrupts running agent in gateway | Agent running a long tool call in Telegram | User sends `/stop` | Agent halts at next iteration boundary; no further tool calls; user notified. *(must bypass both message guards — see AGENTS.md pitfall)* |
| 6 | Dangerous command requires approval | YOLO off, agent calls a guarded shell command | Agent emits tool_call | User receives approval request; `/deny` returns deny-result to model without executing; `/approve` executes. |
| 7 | `/skills install --now` invalidates cache | Skill exists in hub | Run command | Skill installed under `~/.hermes/skills/<cat>/<name>/`; the **next** turn (not current) sees the skill in user-message injection unless `--now` was passed. |
| 8 | Session resume across restart | Session "research-x" persisted | New CLI: `/resume`, pick "research-x" | History restored from SessionDB; trajectory file appended; `/usage` shows accumulated counts. |
| 9 | Gateway allowlist blocks unauthorized user | `TELEGRAM_ALLOWED_USERS=12345`, message from id 67890 | Send Telegram message | Message ignored; warning in `gateway.log`; no `AIAgent` spawned. |
| 10 | Test wrapper enforces hermetic env | Workstation with API keys + 16 cores | Run `scripts/run_tests.sh` | Env vars unset (per script), TZ=UTC, LANG=C.UTF-8, pytest `-n 4`; `tests/conftest.py` redirects HERMES_HOME to tmpdir. |
| 11 | Tool-result format contract | New tool registered via `registry.register(...)` | LLM invokes it | Handler returns a JSON string; non-JSON return -> registry wraps as error; result appended as `{"role":"tool", ...}` message. |
| 12 | Background process notifications respect config | `display.background_process_notifications=error` | Run `terminal(background=true, notify_on_complete=true)` for a successful command | NO notification turn (only fires on non-zero exit). |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| ctr-OQ1 | open question | Exact JSON-RPC error-code map and re-connect semantics for `tui_gateway/server.py` | Wire-format depth belongs to protocols phase; not read directly. |
| ctr-OQ2 | open question | Whether `gateway/platforms/api_server.py` reuses one `AIAgent` per session or pools per request | File ~125KB; not read in this phase's budget (open question carried from arch-OQ5). |
| ctr-OQ3 | open question | CORS / CSP / web-security headers on dashboard `web_server.py` | Not directly read; only `127.0.0.1` bind + ephemeral token observed. |
| ctr-OQ4 | open question | Concrete ordering of shutdown locks (credential pool, scoped platform locks, session flush) | Carried from arch-OQ1; gateway shutdown path not read. |
| ctr-OQ5 | open question | Whether `_isolate_hermes_home` autouse fixture also covers profile multi-instance tests beyond `tests/hermes_cli/test_profiles.py` | Test directory listing not read directly in this phase. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| ctr-CF1 | protocols | TUI JSON-RPC method+event catalog (`prompt.submit`, `slash.exec`, `complete.slash/path`, `session.list/resume`, `approval.respond`, `clarify/sudo/secret.respond`, `gateway.ready`, `command.dispatch`, `tool.start/progress/complete`, `message.delta/complete`) — names enumerated, schemas not. | Wire-format extraction is protocols-phase rubric. |
| ctr-CF2 | protocols | OpenAI-compatible HTTP API request/response shapes for `gateway/platforms/api_server.py`. | Same — schema extraction. |
| ctr-CF3 | protocols | MCP server tool/resource frame schema; ACP server frame schema. | Same. |
| ctr-CF4 | protocols | Persistence schemas: `hermes_state` SQLite tables (FTS5 columns), kanban DB schema, trajectory JSON shape, config.yaml `_config_version` migration table, skin YAML schema, `SKILL.md` frontmatter schema. | Persistence depth belongs in protocols. |
| ctr-CF5 | defect-scan-semantic | Two-message-guard interaction: any new always-reach command must bypass `_pending_messages` AND the `gateway/run.py` interceptor. Concurrency contract violation if either is missed. | Deep concurrency reasoning. |
| ctr-CF6 | defect-scan-semantic | Cache-invariant violations: any slash command that mutates system prompt mid-session breaks prompt caching -> cost spike. | Semantic invariant audit. |
| ctr-CF7 | defect-scan-semantic | Allowlist trapdoor: `GATEWAY_ALLOW_ALL_USERS` / `TEAMS_ALLOW_ALL_USERS` flip a single env var to "open access." Verify no platform adapter ignores allowlist when env is unset. | Authz audit. |
| ctr-CF8 | porting | Three-loader config drift: re-implementations should consolidate to a single config loader with explicit gateway-vs-CLI views. | Porting design decision. |
| ctr-CF9 | porting | Sync agent loop wrapped in async gateway: a re-implementation must decide whether to keep `run_conversation` blocking-by-design or translate to coroutines. | Architectural choice. |
| ctr-CF10 | reimplementation-spec | MVP cut of 18+ messaging platforms: which ship in the first port and which become later phases. | Strategic alignment hook. |
| ctr-CF11 | reimplementation-spec | Decision on whether the OpenAI-compatible HTTP API server is in MVP scope (it duplicates much of MCP/ACP). | Strategic. |

---
