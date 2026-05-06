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

## 2026-05-06 — protocols phase

This section appends concrete schemas extracted from direct reads of `hermes_state.py`, `gateway/session.py`, `gateway/config.py`, and `mcp_serve.py`. It resolves arch-CF3 (persistent schemas).

### `hermes_state.SessionDB` — SQLite, schema_version = 11

*(observed fact — `hermes_state.SCHEMA_SQL`)*

```
CREATE TABLE schema_version(version INTEGER NOT NULL);

CREATE TABLE sessions(
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,           -- 'cli' | 'telegram' | 'discord' | ...
  user_id TEXT,
  model TEXT,
  model_config TEXT,              -- JSON
  system_prompt TEXT,
  parent_session_id TEXT REFERENCES sessions(id),  -- compression-split chain
  started_at REAL NOT NULL,       -- unix epoch
  ended_at REAL,
  end_reason TEXT,                -- 'session_reset' | 'session_switch' | ...
  message_count INTEGER DEFAULT 0,
  tool_call_count INTEGER DEFAULT 0,
  input_tokens INTEGER DEFAULT 0,
  output_tokens INTEGER DEFAULT 0,
  cache_read_tokens INTEGER DEFAULT 0,
  cache_write_tokens INTEGER DEFAULT 0,
  reasoning_tokens INTEGER DEFAULT 0,
  billing_provider TEXT, billing_base_url TEXT, billing_mode TEXT,
  estimated_cost_usd REAL, actual_cost_usd REAL,
  cost_status TEXT, cost_source TEXT, pricing_version TEXT,
  title TEXT,
  api_call_count INTEGER DEFAULT 0
);

CREATE TABLE messages(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL REFERENCES sessions(id),
  role TEXT NOT NULL,             -- 'user' | 'assistant' | 'tool' | 'system'
  content TEXT,                   -- string OR JSON-encoded multipart
  tool_call_id TEXT,
  tool_calls TEXT,                -- JSON
  tool_name TEXT,
  timestamp REAL NOT NULL,
  token_count INTEGER,
  finish_reason TEXT,
  reasoning TEXT,
  reasoning_content TEXT,
  reasoning_details TEXT,
  codex_reasoning_items TEXT,
  codex_message_items TEXT
);

CREATE TABLE state_meta(key TEXT PRIMARY KEY, value TEXT);

CREATE INDEX idx_sessions_source ON sessions(source);
CREATE INDEX idx_sessions_parent ON sessions(parent_session_id);
CREATE INDEX idx_sessions_started ON sessions(started_at DESC);
CREATE INDEX idx_messages_session ON messages(session_id, timestamp);
```

- WAL mode (`PRAGMA journal_mode=WAL`) — concurrent readers + one writer. *(observed fact — file docstring)*
- FTS5 virtual table over `messages.content` for cross-session search. *(observed fact — file docstring + AGENTS.md)*
- `messages` is append-only EXCEPT `replace_messages(session_id, messages)` (called by `/retry`, `/undo`, `/compress`).
- `sessions` is mutable for token totals + lifecycle fields; `parent_session_id` chains preserve compression-split history (linear chain, not branching).
- `end_session(session_id, end_reason)` flips `ended_at`; `reopen_session(session_id)` restores active state for `/resume`.
- Batch runner and RL trajectories are NOT stored here — they remain in JSON / JSONL outside the DB. *(observed fact — file docstring)*

### `gateway/sessions/sessions.json` — Session index

*(observed fact — `gateway/session.py` `SessionStore`, `SessionEntry.to_dict`)*

```
{
  "<session_key>": {
    "session_key": "agent:main:telegram:dm:<chat_id>",
    "session_id": "YYYYMMDD_HHMMSS_<uuid8>",
    "created_at": "<iso8601>", "updated_at": "<iso8601>",
    "display_name": "…",
    "platform": "telegram",  -- Platform enum value
    "chat_type": "dm" | "group" | "channel" | "thread",
    "input_tokens": 0, "output_tokens": 0,
    "cache_read_tokens": 0, "cache_write_tokens": 0,
    "total_tokens": 0, "last_prompt_tokens": 0,
    "estimated_cost_usd": 0.0, "cost_status": "unknown",
    "expiry_finalized": false,
    "suspended": false,
    "resume_pending": false, "resume_reason": null,
    "last_resume_marked_at": null,
    "is_fresh_reset": false,
    "origin": {
      "platform": "telegram", "chat_id": "...", "chat_name": "...",
      "chat_type": "dm", "user_id": "...", "user_name": "...",
      "thread_id": null, "chat_topic": null,
      "user_id_alt": null, "chat_id_alt": null,
      "guild_id": null, "parent_chat_id": null, "message_id": null
    }
  }
}
```

- Atomic write: tempfile in same dir, `f.flush()` + `os.fsync()`, then `utils.atomic_replace(tmp, target)`.
- Mutated on every state change; mtime is the polling driver in `mcp_serve.EventBridge` (200ms cycle, mtime-cached fast path).
- Survives restart fields are `suspended`, `resume_pending`, `resume_reason`, `last_resume_marked_at`, `expiry_finalized`, `is_fresh_reset`.
- Unknown platform values silently skipped on load (forward-compat for plugin platforms).
- Session-key construction is deterministic: see `build_session_key()` patterns in protocols-and-state.md §Event Catalog.

### Legacy JSONL transcript — `gateway/sessions/<session_id>.jsonl`

*(observed fact — `SessionStore.append_to_transcript`, `rewrite_transcript`)*

- One JSON message per line, UTF-8, `ensure_ascii=False`.
- Append-only EXCEPT `/retry`, `/undo`, `/compress` rewrite via `rewrite_transcript` (open-truncate-write — NOT atomic).
- Written redundantly with SQLite. The `skip_db=True` flag is used when the agent already persisted via `_flush_messages_to_session_db()` to prevent duplicate-write bug #860.
- Read fallback: `SessionStore.load_transcript` prefers JSONL when `len(jsonl_messages) > len(db_messages)` to avoid silent truncation of legacy sessions where SQLite has only post-migration deltas.
- Corrupt lines skipped with WARNING.

### Trajectory JSON — `logs/session_*.json`

*(observed fact — runtime-lifecycle.md, AGENTS.md, references in `agent/trajectory.py`, `batch_runner.py`)*

- Whole-conversation snapshot at session end (or on demand): `messages[]`, `model`, `started_at`, `ended_at`, token totals, reasoning where applicable.
- Generated when `save_trajectories=True`.
- Append-only at the file level; rewritten only on explicit edit. Used as RL training corpus by `environments/`, consumed by `trajectory_compressor`, `agent/curator`, `batch_runner`, `rl_cli`.

### Prompt-cache layout (provider-side; protocol-level)

*(observed fact — AGENTS.md, runtime-lifecycle.md)*

- System prompt is **cache-stable** by construction in `agent/prompt_builder.py`.
- Mid-conversation rule: do not alter past context, change toolsets, or rebuild system prompts mid-conversation.
- Only legitimate mutation point: `agent/context_compressor.py` triggered at `compression.threshold` (default 0.85).
- Cache cost tracked via `sessions.cache_read_tokens` + `cache_write_tokens` columns.
- Prompt-cache invariant violations routed to defect-scan-semantic prot-CF2 / arch-CF9.

### Reset policy schema (`SessionResetPolicy`)

*(observed fact — `gateway/config.py`)*

```
mode: "daily" | "idle" | "both" | "none"   (default "both")
at_hour: 0..23                              (default 4, LOCAL TIME)
idle_minutes: int                           (default 1440 = 24h)
notify: bool                                (default True)
notify_exclude_platforms: tuple[str]        (default ("api_server","webhook"))
```

Reset triggers (slash commands): default `["/new", "/reset"]` per `GatewayConfig.reset_triggers`.

### `HomeChannel` schema

*(observed fact — `gateway/config.py`)*

```
{ "platform": "<Platform enum value>", "chat_id": "<str>", "name": "<str>" }
```

Used for cron-job delivery when `deliver="<platform>"` without an explicit chat ID.

### Channel directory — `<HERMES_HOME>/channel_directory.json`

*(observed fact — `mcp_serve._load_channel_directory`)*

- Cached list of `platform → [{id|chat_id, name|display_name, type}]`.
- Consumed by MCP `channels_list` tool. Falls back to `sessions.json` enumeration when missing.
- Mutable; refreshed by gateway as platform memberships change. *(strong inference)*

### MCP OAuth tokens — `tools/mcp_oauth_manager.py`

*(observed fact — state-and-storage.md baseline)*

- Per-MCP-server credentials persisted as JSON under HERMES_HOME.
- Refresh-on-401 reentrancy + clock skew on token expiry are flagged compatibility hazards (see protocols-and-state.md §Compatibility Hazards).

### Open Schema Items (deferred)

- Kanban DB schema (`kanban_db.py`) — not read this phase. *(prot-OQ5)*
- asyncpg / Postgres alternative — extras `[gateway]` includes `asyncpg` but exact migration parity with SQLite is unread. *(prot-OQ3)*
- Approval persistence path on the CLI (allow-always grants) — not read. *(prot-OQ6)*
