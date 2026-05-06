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
| prompt_toolkit interactive REPL | `cli.py` | KawaiiSpinner + `┊` activity feed; banner + response box |
| Ink (React) TUI | `ui-tui/src/app.tsx` | Streaming transcript, prompts, completions; production via `npm run build` |
| Browser dashboard with embedded TUI | `web/` SPA + `hermes_cli/web_server.py` | xterm.js (WebGL + fit + unicode11 addons); 127.0.0.1 only |
| Setup wizard | `hermes_cli/setup.py` (~136KB) | First-run config + provider/keys; OpenClaw migration prompt |
| Tool config menu (curses) | `hermes_cli/curses_ui.py` | Preferred over legacy `simple-term-menu` |
