# Public Surfaces

## 2026-05-06 — architecture phase

Catalog of every boundary the outside world touches. All entries `observed fact` from pyproject.toml, README.md, AGENTS.md, .env.example, docker-compose.yml, and directory listings unless marked otherwise.

### Console scripts (pyproject `[project.scripts]`)

| Command | Module | Purpose |
|---|---|---|
| `hermes` | `hermes_cli.main:main` | Primary user CLI / TUI dispatcher |
| `hermes-agent` | `run_agent:main` | Core agent runner (also reused by gateway) |
| `hermes-acp` | `acp_adapter.entry:main` | Agent Client Protocol adapter for IDEs |

### Top-level executable Python modules (declared as `py-modules`)

`run_agent`, `model_tools`, `toolsets`, `batch_runner`, `trajectory_compressor`, `toolset_distributions`, `cli`, `hermes_constants`, `hermes_state`, `hermes_time`, `hermes_logging`, `rl_cli`, `utils`, `mcp_serve` (MCP server entrypoint).

### `hermes` subcommands (advertised in README; many handler files in `hermes_cli/`)

| Subcommand | Handler module | Notes |
|---|---|---|
| `hermes` (no args) | `cli.HermesCLI` | Interactive REPL |
| `hermes --tui` / `HERMES_TUI=1` | `ui-tui/` Ink + `tui_gateway/` | TypeScript chat surface over stdio JSON-RPC |
| `hermes -p <profile>` | `hermes_cli/profiles.py` | Profile multi-instance |
| `hermes model` | `hermes_cli/model_switch.py`, `models.py` | Provider/model picker |
| `hermes tools` | `hermes_cli/tools_config.py` | Toolset enable/disable |
| `hermes config set` | `hermes_cli/config.py` | Config dotpath setter |
| `hermes setup` | `hermes_cli/setup.py` | Full first-run wizard |
| `hermes gateway setup` | `hermes_cli/gateway.py` | Gateway setup wizard |
| `hermes gateway start` / `hermes gateway run` | `gateway/run.py` | Long-running messaging service |
| `hermes claw migrate` | `hermes_cli/claw.py` | OpenClaw import (dry-run / preset / overwrite) |
| `hermes update` | `hermes_cli/relaunch.py` (likely) | Self-update |
| `hermes doctor` | `hermes_cli/doctor.py` | Diagnostics |
| `hermes logs` | `hermes_cli/logs.py` | `--follow --level --session` |
| `hermes dashboard` | `hermes_cli/web_server.py` | FastAPI SPA + `/api/pty` WS |
| `hermes whatsapp` | `gateway/platforms/whatsapp.py` (Baileys) | WhatsApp pairing |
| `hermes skills install official/<category>/<skill>` | `tools/skills_hub.py` | Optional-skill activation |
| `hermes <plugin> ...` | `hermes_cli/plugins.py` | Plugin-registered argparse subtree |
| `hermes voice` | `hermes_cli/voice.py` | Voice mode |
| `hermes webhook` | `hermes_cli/webhook.py` | Webhook receiver mgmt |
| `hermes cron` | `hermes_cli/cron.py` | Cron job mgmt |
| `hermes uninstall` | `hermes_cli/uninstall.py` | Uninstall |
| `hermes kanban` | `hermes_cli/kanban.py` | Kanban TUI |

### Slash commands (in-conversation; central registry `hermes_cli/commands.py`)

Documented in AGENTS.md / README:

| Slash | Surface | Notes |
|---|---|---|
| `/new`, `/reset` | CLI + Gateway | Start fresh conversation |
| `/model [provider:model]` | CLI + Gateway | Switch model |
| `/personality [name]` | CLI + Gateway | Switch persona |
| `/retry`, `/undo` | CLI + Gateway | Replay/rewind last turn |
| `/compress`, `/usage`, `/insights [--days N]` | CLI + Gateway | Context management |
| `/skills`, `/<skill-name>` | CLI + Gateway | Browse / invoke skills |
| `/stop` | Gateway | Cancel running agent |
| `/queue`, `/status` | Gateway | Inspect message queue |
| `/approve`, `/deny` | Gateway | Approve/deny pending tool |
| `/platforms` | CLI | Platform-specific status |
| `/sethome` | Gateway | Set home channel |
| `/help` | both | Help |
| `/skin <name>` | CLI | Activate skin (deferred invalidation by default; `--now` for immediate) |
| `/copy`, `/paste`, `/clear`, `/quit`, `/resume` | TUI/CLI | Local handlers |
| `/<plugin-cmd>` | CLI + Gateway | Plugin-registered slash |

Gateway-specific dispatch sources: `GATEWAY_KNOWN_COMMANDS` frozenset, `gateway_help_lines()` (Help), `telegram_bot_commands()` (Telegram BotCommand menu), `slack_subcommand_map()` (Slack `/hermes` subcommands).

### Network / RPC interfaces

| Interface | Module | Notes |
|---|---|---|
| OpenAI-compatible HTTP API | `gateway/platforms/api_server.py` | Off unless `API_SERVER_KEY` + `API_SERVER_HOST` set |
| Dashboard HTTP + `/api/pty` WebSocket | `hermes_cli/web_server.py` | Default 127.0.0.1; ephemeral `_SESSION_TOKEN`; PTY frames; `\x1b[RESIZE:cols;rows]` resize protocol |
| MCP server (stdio JSON-RPC) | `mcp_serve.py` | `[mcp]` extra |
| ACP server (stdio JSON-RPC) | `acp_adapter/entry.py` | `[acp]` extra; for Zed/VS Code/JetBrains |
| TUI JSON-RPC (stdio newline-delimited) | `tui_gateway/server.py` | Methods include `prompt.submit`, `slash.exec`, `complete.slash`, `complete.path`, `session.list/resume`, `approval.respond`, `clarify/sudo/secret.respond`, `gateway.ready`, `command.dispatch`, `tool.start/progress/complete`, `message.delta/complete` |
| Telegram (long-poll OR webhook) | `gateway/platforms/telegram.py` | `TELEGRAM_WEBHOOK_URL` switches modes |
| Discord (gateway) | `gateway/platforms/discord.py` | `discord.py[voice]` |
| Slack (Socket Mode + slash) | `gateway/platforms/slack.py` | `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` |
| Matrix client | `gateway/platforms/matrix.py` | `mautrix[encryption]`; Linux only by default |
| WhatsApp Baileys bridge | `gateway/platforms/whatsapp.py` | `hermes whatsapp` to pair |
| Email (IMAP+SMTP) | `gateway/platforms/email.py` | `EMAIL_*` env vars |
| SMS | `gateway/platforms/sms.py` | `[sms]` extra |
| Mattermost | `gateway/platforms/mattermost.py` | |
| Microsoft Teams (Bot Framework) | (in `gateway/platforms/`) | `TEAMS_PORT` 3978 |
| Home Assistant | `gateway/platforms/homeassistant.py` | `[homeassistant]` extra |
| BlueBubbles | `gateway/platforms/bluebubbles.py` | iMessage bridge |
| DingTalk / WeCom / Weixin / Feishu / QQ / Yuanbao | `gateway/platforms/{dingtalk,wecom,weixin,feishu,qqbot,yuanbao}.py` | Region-specific |
| Webhook receiver | `gateway/platforms/webhook.py` | Generic |
| Signal (with rate-limit module) | `gateway/platforms/signal.py` + `signal_rate_limit.py` | |
| RL API server | `RL_API_URL` (default `http://localhost:8080`) | `[rl]` extra |
| Browserbase (cloud browser) | `tools/browser_tool.py` etc. | `BROWSERBASE_API_KEY` + project ID |
| MCP OAuth | `tools/mcp_oauth.py`, `mcp_oauth_manager.py` | Per-server credentials |
| Skills Hub (GitHub) | `tools/skills_hub.py`, `hermes_cli/skills_hub.py` | `GITHUB_TOKEN` for rate limits |

### File formats / persisted artifacts

| Path / Format | Module | Notes |
|---|---|---|
| `~/.hermes/config.yaml` | `hermes_cli/config.py` (`DEFAULT_CONFIG`) | Versioned `_config_version`; deep-merge migration |
| `~/.hermes/.env` | `hermes_cli/env_loader.py` | Secrets only |
| Session DB (SQLite + FTS5) | `hermes_state.SessionDB` | Per-profile |
| Session trajectory JSON | `agent/trajectory.py` | `logs/session_YYYYMMDD_HHMMSS_UUID.json` |
| Logs | `hermes_logging.setup_logging()` | `agent.log`, `errors.log`, `gateway.log` |
| Skills tree | `~/.hermes/skills/<category>/<skill>/SKILL.md` | YAML frontmatter |
| Skin YAML | `~/.hermes/skins/<name>.yaml` | Inherits from `default` skin |
| Plugins tree | `~/.hermes/plugins/`, `./.hermes/plugins/`, pip entry points | `register(ctx)` ABI |
| Kanban DB | `hermes_cli/kanban_db.py` | SQLite |
| Honcho config | `~/.honcho/config.json` | External |
| Qwen OAuth | `~/.qwen/oauth_creds.json` | External |
| OpenClaw import source | `~/.openclaw/...` | Read-only on migrate |

### User-facing screens / workflows

| Surface | Module | Notes |
|---|---|---|
| prompt_toolkit interactive REPL | `cli.py` | KawaiiSpinner + activity feed; banner + response box |
| Ink (React) TUI | `ui-tui/src/app.tsx` | Streaming transcript, prompts, completions; production via `npm run build` |
| Browser dashboard with embedded TUI | `web/` SPA + `hermes_cli/web_server.py` | xterm.js (WebGL + fit + unicode11 addons); 127.0.0.1 only |
| Setup wizard | `hermes_cli/setup.py` (~136KB) | First-run config + provider/keys; OpenClaw migration prompt |
| Tool config menu (curses) | `hermes_cli/curses_ui.py` | Preferred over legacy `simple-term-menu` |

## 2026-05-06 — contracts phase

Resolves arch-CF1 (full slash-command catalog) and adds MCP/ACP RPC name inventory plus per-platform gateway commands. All entries `observed fact` from `hermes_cli/commands.py` `COMMAND_REGISTRY` parse, AGENTS.md, .env.example, gateway/platforms listing, and `mcp_serve.py`/`acp_adapter/entry.py` references in AGENTS.md unless marked.

### Full slash-command catalog (resolves arch-CF1)

From `hermes_cli/commands.py` `COMMAND_REGISTRY`. Categories: Session, Configuration, Tools & Skills, Info, Exit. Each row's `cli_only` / `gateway_only` / `gateway_config_gate` field controls visibility.

| Slash | Category | Description (from registry) | Surface |
|---|---|---|---|
| `/new` | Session | Start a new session (fresh session ID + history) | CLI + Gateway |
| `/clear` | Session | Clear screen and start a new session | CLI/TUI |
| `/redraw` | Session | Force a full UI repaint (recovers from terminal drift) | TUI/CLI |
| `/history` | Session | Show conversation history | CLI |
| `/save` | Session | Save the current conversation | CLI |
| `/retry` | Session | Retry the last message (resend to agent) | CLI + Gateway |
| `/undo` | Session | Remove the last user/assistant exchange | CLI + Gateway |
| `/title` | Session | Set a title for the current session | CLI |
| `/branch` | Session | Branch the current session (explore a different path) | CLI |
| `/compress` | Session | Manually compress conversation context | CLI + Gateway |
| `/rollback` | Session | List or restore filesystem checkpoints | CLI |
| `/snapshot` | Session | Create or restore state snapshots of Hermes config/state | CLI |
| `/stop` | Session | Kill all running background processes | Gateway |
| `/approve` | Session | Approve a pending dangerous command | CLI + Gateway |
| `/deny` | Session | Deny a pending dangerous command | CLI + Gateway |
| `/background` | Session | Run a prompt in the background | CLI |
| `/agents` | Session | Show active agents and running tasks | CLI + Gateway |
| `/queue` | Session | Queue a prompt for the next turn (doesn't interrupt) | Gateway |
| `/steer` | Session | Inject a message after the next tool call without interrupting | CLI + Gateway |
| `/status` | Session | Show session info | Gateway |
| `/profile` | Info | Show active profile name and home directory | CLI + Gateway |
| `/sethome` | Session | Set this chat as the home channel | Gateway |
| `/resume` | Session | Resume a previously-named session | CLI |
| `/config` | Configuration | Show current configuration | CLI |
| `/model` | Configuration | Switch model for this session | CLI + Gateway |
| `/gquota` | Info | Show Google Gemini Code Assist quota usage | CLI + Gateway |
| `/personality` | Configuration | Set a predefined personality | CLI + Gateway |
| `/statusbar` | Configuration | Toggle the context/model status bar | CLI |
| `/verbose` | Configuration | Cycle tool progress display: off -> new -> all -> verbose (gated via `gateway_config_gate=display.tool_progress_command`) | CLI (gateway when gated) |
| `/footer` | Configuration | Toggle gateway runtime-metadata footer on final replies | Gateway |
| `/yolo` | Configuration | Toggle YOLO mode (skip all dangerous command approvals) | CLI + Gateway |
| `/reasoning` | Configuration | Manage reasoning effort and display | CLI + Gateway |
| `/fast` | Configuration | Toggle fast mode — OpenAI Priority Processing / Anthropic Fast | CLI + Gateway |
| `/skin` | Configuration | Show or change the display skin/theme | CLI |
| `/indicator` | Configuration | Pick the TUI busy-indicator style | TUI |
| `/voice` | Configuration | Toggle voice mode | CLI |
| `/busy` | Configuration | Control what Enter does while Hermes is working | CLI |
| `/tools` | Tools & Skills | Manage tools: `/tools [list\|disable\|enable] [name...]` | CLI + Gateway |
| `/toolsets` | Tools & Skills | List available toolsets | CLI + Gateway |
| `/skills` | Tools & Skills | Search, install, inspect, or manage skills | CLI + Gateway |
| `/cron` | Tools & Skills | Manage scheduled tasks | CLI + Gateway |
| `/curator` | Tools & Skills | Background skill maintenance (status, run, pin, archive) | CLI + Gateway |
| `/kanban` | Tools & Skills | Multi-profile collaboration board (tasks, links, comments) | CLI + Gateway |
| `/reload` | Tools & Skills | Reload .env variables into the running session | CLI |
| `/reload-mcp` | Tools & Skills | Reload MCP servers from config | CLI + Gateway |
| `/reload-skills` | Tools & Skills | Re-scan ~/.hermes/skills/ for newly installed or removed skills | CLI + Gateway |
| `/browser` | Tools & Skills | Connect browser tools to your live Chrome via CDP | CLI |
| `/plugins` | Tools & Skills | List installed plugins and their status | CLI + Gateway |
| `/commands` | Info | Browse all commands and skills (paginated) | CLI |
| `/help` | Info | Show available commands | CLI + Gateway |
| `/restart` | Session | Gracefully restart the gateway after draining active runs | Gateway |
| `/usage` | Info | Show token usage and rate limits for the current session | CLI + Gateway |
| `/insights` | Info | Show usage insights and analytics | CLI + Gateway |
| `/platforms` | Info | Show gateway/messaging platform status | Gateway |
| `/copy` | Info | Copy the last assistant response to clipboard | CLI |
| `/paste` | Info | Attach clipboard image from your clipboard | CLI |
| `/image` | Info | Attach a local image file for your next prompt | CLI |
| `/update` | Info | Update Hermes Agent to the latest version | CLI |
| `/debug` | Info | Upload debug report (system info + logs) and get shareable link | CLI + Gateway |
| `/quit` | Exit | Exit the CLI | CLI |

*Plugin-contributed slashes are wired in dynamically; not enumerable here.*

### MCP-exposed RPC names (`mcp_serve.py`)

MCP server is enabled via the `[mcp]` extra and follows the Model Context Protocol spec. Methods directly observed in references:

- Standard MCP framing: `initialize`, `tools/list`, `tools/call`, `resources/list`, `resources/read`, `prompts/list`, `prompts/get`, `logging/setLevel`, plus notification `notifications/cancelled`, `notifications/initialized`, `notifications/progress`. *(strong inference — MCP spec; not directly read in budget)*
- Tools surfaced: every entry registered in `tools.registry.registry` schema is exposed as an MCP tool. The full registry is auto-discovered at `mcp_serve.py` import time. *(observed fact — AGENTS.md "Adding New Tools" describes the registry as the single source of truth)*
- *Carry-forward (ctr-CF3):* exact MCP method enumeration and per-method schema not extracted; deferred to protocols phase.

### ACP server (`acp_adapter/entry.py`)

- ACP frames per agent-client-protocol spec; spawns/manages an `AIAgent` per session. *(observed fact — AGENTS.md, pyproject `[acp]` extra)*
- *Carry-forward (ctr-CF3):* ACP method/event enumeration deferred to protocols phase.

### TUI JSON-RPC method+event catalog (`tui_gateway/server.py`)

Methods (Ink -> Python):

- `prompt.submit` — submit a chat prompt (returns nothing; events stream)
- `slash.exec` — run slash command in `_SlashWorker` subprocess
- `command.dispatch` — fallback path for non-built-in commands
- `complete.slash` — autocomplete slash names
- `complete.path` — path completion in composer
- `session.list` — list known sessions
- `session.resume` — resume by id/name
- `approval.respond` — respond to dangerous-command prompt
- `clarify.respond` — respond to clarifying-question prompt
- `sudo.respond` — respond to sudo password prompt
- `secret.respond` — respond to secret-input prompt (masked)

Events (Python -> Ink):

- `gateway.ready` — initial handshake (includes skin data)
- `message.delta` / `message.complete` — streaming chat output
- `tool.start` / `tool.progress` / `tool.complete` — tool activity
- `approval.request` / `clarify.request` / `sudo.request` / `secret.request` — interactive prompts

*Names enumerated; per-method JSON schemas deferred via ctr-CF1 to protocols phase. (observed fact — AGENTS.md "TUI Architecture" key surfaces table)*

### Gateway commands per platform

Two derived dispatch sources from `hermes_cli/commands.py`:

- **Telegram BotCommand menu** — `telegram_bot_commands()` emits the slashes that appear in the Telegram client's command menu. Subset of `GATEWAY_KNOWN_COMMANDS`. *(observed fact — AGENTS.md slash-command registry section)*
- **Slack `/hermes` subcommand mapping** — `slack_subcommand_map()` registers each gateway slash as a Slack subcommand under the `/hermes` slash. *(observed fact — AGENTS.md)*
- **`GATEWAY_KNOWN_COMMANDS`** — frozenset of canonical names available in the gateway dispatch path. Always includes `gateway_config_gate`d CLI-only commands so dispatch is possible; help/menus only show them when the gate is open.

Per-platform notes (from architecture-map and .env.example):

| Platform | Trigger surface | Allowlist env | Home-channel env |
|---|---|---|---|
| Telegram | Long-poll or webhook (`TELEGRAM_WEBHOOK_URL`); BotCommand menu populated from `telegram_bot_commands()` | `TELEGRAM_ALLOWED_USERS` | `TELEGRAM_HOME_CHANNEL` (+ `_NAME`) |
| Slack | Socket Mode + `/hermes <subcmd>` via `slack_subcommand_map()` | `SLACK_ALLOWED_USERS` | n/a |
| Discord | Gateway connection (discord.py) | (per-platform, via gateway config) | n/a |
| WhatsApp | Baileys bridge; pair via `hermes whatsapp` | `WHATSAPP_ALLOWED_USERS` | n/a |
| Email | IMAP poll + SMTP send | `EMAIL_ALLOWED_USERS` | `EMAIL_HOME_ADDRESS` |
| Microsoft Teams | Bot Framework on `TEAMS_PORT=3978` | `TEAMS_ALLOWED_USERS` (or `TEAMS_ALLOW_ALL_USERS=true`) | `TEAMS_HOME_CHANNEL` (+ `_NAME`) |
| Generic gateway-wide | n/a | `GATEWAY_ALLOW_ALL_USERS=false` (default = deny) | n/a |

*Other adapters (Signal, Matrix, Mattermost, Home Assistant, BlueBubbles, Webhook, DingTalk, WeCom, Weixin, Feishu, QQ, Yuanbao) follow the same pattern — adapter file in `gateway/platforms/` + per-platform allowlist; not enumerated exhaustively in this read budget.*
