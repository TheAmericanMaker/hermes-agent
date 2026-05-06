# State and Storage

## 2026-05-06 — architecture phase

All entries `observed fact` unless marked.

### Profile mechanism

- `_apply_profile_override()` in `hermes_cli/main.py` sets `HERMES_HOME` env var BEFORE any module imports.
- `hermes_constants.get_hermes_home()` returns the active profile's home; **never** hardcode `Path.home() / ".hermes"`.
- `hermes_constants.display_hermes_home()` returns user-facing path string.
- `_get_profiles_root()` returns `Path.home() / ".hermes" / "profiles"` (HOME-anchored, NOT HERMES_HOME-anchored, so `hermes -p coder profile list` works regardless of active profile).

### File-on-disk inventory (under `HERMES_HOME`, default `~/.hermes/`)

| Path | Owner module | Format | Purpose |
|---|---|---|---|
| `config.yaml` | `hermes_cli/config.py` (`DEFAULT_CONFIG`) | YAML | Settings; versioned `_config_version`; deep-merged on load |
| `.env` | `hermes_cli/env_loader.py` | KEY=VALUE | Secrets only (API keys, tokens, passwords) |
| `logs/agent.log` | `hermes_logging.setup_logging()` | Plain log | INFO+ |
| `logs/errors.log` | `hermes_logging` | Plain log | WARNING+ |
| `logs/gateway.log` | `hermes_logging` | Plain log | Gateway-specific |
| `logs/session_YYYYMMDD_HHMMSS_UUID.json` | `agent/trajectory.py` | Trajectory JSON | Conversation replay format |
| `sessions.db` (or similar) | `hermes_state.SessionDB` | SQLite + FTS5 | Cross-session search & resume |
| `kanban.db` (or similar) | `hermes_cli/kanban_db.py` | SQLite | Kanban tracker |
| `skills/<category>/<skill>/SKILL.md` | `agent/skill_utils.py`, `tools/skills_hub.py` | Markdown + YAML frontmatter | User-installed skills |
| `skills/openclaw-imports/...` | `hermes_cli/claw.py` | mirror | OpenClaw migration target |
| `skins/<name>.yaml` | `hermes_cli/skin_engine.py` | YAML | User-installed skins |
| `plugins/<name>/` | `hermes_cli/plugins.py` | Python pkg | User plugins |
| `profiles/<name>/` | `hermes_cli/profiles.py` | Mirror of HOME tree | Per-profile state |
| Browser session state | `tools/browser_tool.py` etc. | Browserbase-managed | Remote browser session id |
| MCP OAuth tokens | `tools/mcp_oauth_manager.py` | JSON | Per-MCP-server credentials |
| Sticker cache | `gateway/sticker_cache.py` | Files | Telegram/Discord stickers |
| WhatsApp identity | `gateway/whatsapp_identity.py` | JSON | Baileys keys |

### External / cross-tool state

| Path | Source |
|---|---|
| `~/.openclaw/...` | OpenClaw legacy install (read on `hermes claw migrate`) |
| `~/.honcho/config.json` | Honcho memory provider |
| `~/.qwen/oauth_creds.json` | Qwen OAuth provider |
| `~/.local/bin/hermes` | Symlink installed by `setup-hermes.sh` |
| Playwright browsers | Container: `/opt/hermes/.playwright`; host: per-OS default |
| Modal credentials | Modal CLI (`modal setup`) |
| Daytona credentials | Daytona CLI |
| Weights & Biases | `~/.netrc` or `WANDB_API_KEY` |

### Environment variables (extracted from `.env.example`)

#### Provider keys

`OPENROUTER_API_KEY`, `GOOGLE_API_KEY`, `GEMINI_API_KEY`, `GEMINI_BASE_URL`, `OLLAMA_API_KEY`, `OLLAMA_BASE_URL`, `GLM_API_KEY`, `GLM_BASE_URL`, `KIMI_API_KEY`, `KIMI_BASE_URL`, `KIMI_CN_API_KEY`, `ARCEEAI_API_KEY`, `ARCEE_BASE_URL`, `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_CN_API_KEY`, `MINIMAX_CN_BASE_URL`, `OPENCODE_ZEN_API_KEY`, `OPENCODE_ZEN_BASE_URL`, `OPENCODE_GO_API_KEY`, `OPENCODE_GO_BASE_URL`, `HF_TOKEN`, `HERMES_QWEN_BASE_URL`, `XIAOMI_API_KEY`, `XIAOMI_BASE_URL`, `NOUS_API_KEY` (implicit; redacted in tests).

#### Tool keys

`EXA_API_KEY`, `PARALLEL_API_KEY`, `FIRECRAWL_API_KEY`, `FAL_KEY`, `HONCHO_API_KEY`, `BROWSERBASE_API_KEY`, `BROWSERBASE_PROJECT_ID`, `BROWSERBASE_PROXIES`, `BROWSERBASE_ADVANCED_STEALTH`, `BROWSER_SESSION_TIMEOUT`, `BROWSER_INACTIVITY_TIMEOUT`, `VOICE_TOOLS_OPENAI_KEY`, `STT_GROQ_MODEL`, `STT_OPENAI_MODEL`, `GROQ_BASE_URL`, `STT_OPENAI_BASE_URL`, `GROQ_API_KEY`, `GITHUB_TOKEN`, `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY_PATH`, `GITHUB_APP_INSTALLATION_ID`, `TINKER_API_KEY`, `WANDB_API_KEY`, `RL_API_URL`.

#### Terminal / sandbox

`TERMINAL_ENV`, `HERMES_DOCKER_BINARY`, `TERMINAL_DOCKER_IMAGE`, `TERMINAL_SINGULARITY_IMAGE`, `TERMINAL_MODAL_IMAGE`, `TERMINAL_CWD` (deprecated—use `terminal.cwd`), `TERMINAL_TIMEOUT`, `TERMINAL_LIFETIME_SECONDS`, `TERMINAL_SSH_HOST`, `TERMINAL_SSH_USER`, `TERMINAL_SSH_PORT`, `TERMINAL_SSH_KEY`, `SUDO_PASSWORD`.

#### Messaging / gateway

`SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`, `SLACK_ALLOWED_USERS`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, `TELEGRAM_HOME_CHANNEL`, `TELEGRAM_HOME_CHANNEL_NAME`, `TELEGRAM_WEBHOOK_URL`, `TELEGRAM_WEBHOOK_PORT`, `TELEGRAM_WEBHOOK_SECRET`, `WHATSAPP_ENABLED`, `WHATSAPP_ALLOWED_USERS`, `EMAIL_ADDRESS`, `EMAIL_PASSWORD`, `EMAIL_IMAP_HOST`, `EMAIL_IMAP_PORT`, `EMAIL_SMTP_HOST`, `EMAIL_SMTP_PORT`, `EMAIL_POLL_INTERVAL`, `EMAIL_ALLOWED_USERS`, `EMAIL_HOME_ADDRESS`, `GATEWAY_ALLOW_ALL_USERS`, `MESSAGING_CWD` (deprecated), `TEAMS_CLIENT_ID`, `TEAMS_CLIENT_SECRET`, `TEAMS_TENANT_ID`, `TEAMS_ALLOWED_USERS`, `TEAMS_ALLOW_ALL_USERS`, `TEAMS_HOME_CHANNEL`, `TEAMS_HOME_CHANNEL_NAME`, `TEAMS_PORT` (3978), `API_SERVER_KEY`, `API_SERVER_HOST`, `HERMES_HUMAN_DELAY_MODE`, `HERMES_HUMAN_DELAY_MIN_MS`, `HERMES_HUMAN_DELAY_MAX_MS`.

#### Compression / display / debug

`CONTEXT_COMPRESSION_ENABLED`, `CONTEXT_COMPRESSION_THRESHOLD`, `WEB_TOOLS_DEBUG`, `VISION_TOOLS_DEBUG`, `MOA_TOOLS_DEBUG`, `IMAGE_TOOLS_DEBUG`, `HERMES_BACKGROUND_NOTIFICATIONS`.

#### Profile / runtime

`HERMES_HOME` (canonical), `HERMES_TUI`, `HERMES_UID`, `HERMES_GID`, `HERMES_WEB_DIST`, `PLAYWRIGHT_BROWSERS_PATH`, `PYTHONUNBUFFERED`, `PATH`.

### Generated build artifacts

| Artifact | Producer |
|---|---|
| `hermes_cli/web_dist/**` | `web/` Vite build, packaged via `[tool.setuptools.package-data]` |
| `node_modules/@hermes/ink` (built TUI) | `ui-tui/` `npm run build` then symlink + `npm install --omit=dev` |
| `dist/*.whl` (Python wheel) | `setuptools.build_meta` |
| Docker image `nousresearch/hermes-agent:{latest,<tag>}` | `Dockerfile` + `.github/workflows/docker-publish.yml` |
| Docusaurus site | `website/` + `.github/workflows/deploy-site.yml` |
| Skills index | `.github/workflows/skills-index.yml` |

### Test isolation

`tests/conftest.py` autouse `_isolate_hermes_home` fixture redirects `HERMES_HOME` to a temp dir per test. CI also unsets every `*_API_KEY`/`*_TOKEN`, sets TZ=UTC and LANG=C.UTF-8, and pins `-n 4` xdist workers via `scripts/run_tests.sh`.
