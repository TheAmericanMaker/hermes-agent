# Config Model

## 2026-05-06 — architecture phase

### Sources & precedence (low -> high)

1. `DEFAULT_CONFIG` baked into `hermes_cli/config.py` (versioned via `_config_version`; deep-merged on load).
2. `~/.hermes/config.yaml` (user YAML; merged over defaults).
3. `~/.hermes/.env` (secrets only — API keys, tokens, passwords). Loaded by `hermes_cli/env_loader.py` via `python-dotenv`.
4. Process environment variables (env-only override surface; some bridged to/from config.yaml — e.g. `terminal.cwd` -> `TERMINAL_CWD` for child tools).
5. CLI flags / subcommand args (highest precedence at runtime).
6. *Plugin-provided overrides* registered via `register(ctx)` ABI (effective for the active session).

### Three config loaders (NOT one)

| Loader | Used by | Behavior |
|---|---|---|
| `load_cli_config()` (top-level `cli.py`) | Interactive CLI mode | Hardcoded CLI defaults + user YAML |
| `load_config()` (`hermes_cli/config.py`) | `hermes tools`, `hermes setup`, most subcommands | `DEFAULT_CONFIG` deep-merge + user YAML |
| Direct YAML load (`gateway/run.py` + `gateway/config.py`) | Gateway runtime | Reads user YAML raw |

If a new key is visible to CLI but not gateway (or vice versa), the wrong loader is in use. **(portability hazard — three sources of truth, AGENTS.md explicitly warns of this)**

### Versioning

- `_config_version` integer at top of `DEFAULT_CONFIG`.
- Bumped only when *active migration* needed (key rename, structural change). Adding a new key with a default does not require a bump.
- Migration applied on `load_config()`: deep-merge handles additive changes automatically.

### Adding config keys

- New `config.yaml` option -> add to `DEFAULT_CONFIG`. Bump version only if migrating.
- New secret -> add to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py` with metadata: `description`, `prompt`, `url`, `password` (bool), `category` (`provider`/`tool`/`messaging`/`setting`).
- Non-secret settings (timeouts, thresholds, feature flags, paths, display preferences) belong in `config.yaml`, **not** `.env`.

### Working-directory model

- CLI: `os.getcwd()`.
- Messaging: `terminal.cwd` from `config.yaml`. Gateway bridges to `TERMINAL_CWD` env var for child tools.
- `MESSAGING_CWD` env var: **deprecated** (warning printed if set in `.env`).
- `TERMINAL_CWD` in `.env`: **deprecated**; canonical is `terminal.cwd` in `config.yaml`.

### Feature flags / config dotpaths visible in AGENTS.md & .env.example

| Dotpath / Env | Default | Effect |
|---|---|---|
| `display.skin` | `default` | Skin engine activates this skin |
| `display.background_process_notifications` (env: `HERMES_BACKGROUND_NOTIFICATIONS`) | `all` | Verbosity of background terminal-process watcher messages: `all`/`result`/`error`/`off` |
| `display.tool_progress_command` | (config-gated) | Gates a CLI-only command for gateway use |
| `compression.enabled` (env: `CONTEXT_COMPRESSION_ENABLED`) | true | Enable auto-compression |
| `compression.threshold` (env: `CONTEXT_COMPRESSION_THRESHOLD`) | 0.85 | Trigger when conversation reaches 85% of context limit |
| `compression.summary_model` | `google/gemini-3-flash-preview` | Model used to summarize for compression |
| `terminal.backend` (env: `TERMINAL_ENV`) | `local` | One of `local`/`docker`/`singularity`/`modal`/`ssh`/`daytona` |
| `terminal.cwd` (env: `TERMINAL_CWD`, deprecated) | (per-backend default) | Terminal working dir |
| `model.default` | (set during setup) | Default LLM |
| `memory.provider` | (per setup) | Memory backend selection: honcho/mem0/supermemory/byterover/hindsight/holographic/openviking/retaindb |
| `stt.provider` | `local` | STT priority `local > groq > openai` |
| `stt.model` (env: `STT_GROQ_MODEL`, `STT_OPENAI_MODEL`) | per-provider | STT model |
| `gateway.timeout` (bridged from config.yaml to `gateway_timeout` env var) | n/a | Gateway timeout |
| `skills.config.<key>` | per skill | Per-skill config (frontmatter `metadata.hermes.config`) |
| `BROWSERBASE_PROXIES` | `true` | Residential proxies on for browser tool |
| `BROWSERBASE_ADVANCED_STEALTH` | `false` | Advanced stealth (Scale Plan only) |
| `WEB_TOOLS_DEBUG`, `VISION_TOOLS_DEBUG`, `MOA_TOOLS_DEBUG`, `IMAGE_TOOLS_DEBUG` | `false` | Per-toolset debug logging |
| `GATEWAY_ALLOW_ALL_USERS` | `false` (deny) | Skip allowlist guard — dangerous |
| `TEAMS_ALLOW_ALL_USERS` | `false` | Same for Teams |
| `HERMES_HUMAN_DELAY_MODE` | `off` | `off`/`natural`/`custom` pacing for messaging |
| `HERMES_HUMAN_DELAY_MIN_MS` | `800` | Custom-mode min delay |
| `HERMES_HUMAN_DELAY_MAX_MS` | `2500` | Custom-mode max delay |
| `BROWSER_SESSION_TIMEOUT` | `300` | Browser session timeout (s) |
| `BROWSER_INACTIVITY_TIMEOUT` | `120` | Browser auto-cleanup |
| `TERMINAL_TIMEOUT` | `60` | Default command timeout (s) |
| `TERMINAL_LIFETIME_SECONDS` | `300` | Cleanup inactive envs after N seconds |

### Cache-aware mutation rule (prompt-cache invariant)

Slash commands that mutate system-prompt state (skills, tools, memory) MUST default to **deferred invalidation** (effective next session); opt-in `--now` flag for immediate invalidation. Canonical pattern: `/skills install --now`. *(observed fact — AGENTS.md "Important Policies")*

### CommandDef config gating (`hermes_cli/commands.py`)

- `gateway_config_gate` field: dotpath into config.yaml. When set on a `cli_only` command, the command becomes available in the gateway iff the config value is truthy.
- `GATEWAY_KNOWN_COMMANDS` always includes config-gated commands so the gateway can dispatch them; help/menus only show them when the gate is open.

### Plugin / memory-provider config

- Plugins: register via `register(ctx)` in `~/.hermes/plugins/<name>/`, `./.hermes/plugins/<name>/`, or pip entry points. Discovery is a side effect of importing `model_tools.py`.
- Memory providers: only the **active** provider's CLI commands appear in `hermes --help`; other providers' commands are hidden until the user switches `memory.provider`.
- The May 2026 "plugins MUST NOT modify core files" rule means plugin-specific argparse logic must not be hardcoded into `hermes_cli/main.py` etc. — it must be wired via the generic plugin surface.

## 2026-05-06 — contracts phase

Resolves contract-phase config-model rubric: explicit precedence resolution table (which loader wins per source), feature-flag inventory with default-vs-override origins, and per-setting defaults sourced from `DEFAULT_CONFIG` (via AGENTS.md) and `.env.example`.

### Config sources & precedence — full resolution table

For a key `K`, the *effective* value at runtime is computed as the highest-precedence source that defines `K`. Bridged keys (config.yaml <-> env var) have a single canonical home; the other side is a backwards-compat mirror.

| Rank (low -> high) | Source | Read by | Notes |
|---|---|---|---|
| 1 | `DEFAULT_CONFIG` literal in `hermes_cli/config.py` | `load_config()` | Versioned via `_config_version`. |
| 2 | `~/.hermes/config.yaml` | `load_config()`, `load_cli_config()`, `gateway/config.py` direct YAML load | Deep-merged over defaults. **(PH — 3 loaders, see arch-map)** |
| 3 | `~/.hermes/.env` | `hermes_cli/env_loader.py` (via python-dotenv) | Secrets only by policy. |
| 4 | Process env at fork time (incl. shell exports) | All Python modules | Some keys ALSO bridge from config.yaml (e.g. `terminal.cwd` -> `TERMINAL_CWD`) for child-tool consumption. |
| 5 | CLI flags / argparse args | `hermes_cli/main.py` dispatcher | E.g. `-p <profile>` mutates `HERMES_HOME` before any module imports. |
| 6 | Plugin overrides via `register(ctx)` | `model_tools.discover_plugins()` | Effective for the active session only. |

**Profile resolution timing.** `_apply_profile_override()` in `hermes_cli/main.py` MUST set `HERMES_HOME` BEFORE any module that calls `get_hermes_home()` is imported — otherwise paths cache to the wrong profile. *(observed fact — AGENTS.md "Profiles: Multi-Instance Support")*

### Feature flag inventory (with origin & default)

Grouped by area. Origin column: `cfg` = config.yaml dotpath; `env` = .env or process env; `cfg+env` = canonical in config.yaml with env mirror.

| Area | Flag | Origin | Default | Effect |
|---|---|---|---|---|
| Display | `display.skin` | cfg | `default` | Active skin/theme |
| Display | `display.background_process_notifications` | cfg+env (`HERMES_BACKGROUND_NOTIFICATIONS`) | `all` | Background process notification verbosity (`all`/`result`/`error`/`off`) |
| Display | `display.tool_progress_command` | cfg | (unset) | When truthy, opens `/verbose` to gateway via `gateway_config_gate` |
| Display | `statusbar` (toggled via `/statusbar`) | cfg (set by command) | on | Context/model status bar |
| Display | `footer` (toggled via `/footer`) | cfg (set by command) | (per-setup) | Gateway runtime-metadata footer on final replies |
| Display | `indicator` (TUI busy indicator) | cfg | (skin default) | Set via `/indicator` |
| Compression | `compression.enabled` | cfg+env (`CONTEXT_COMPRESSION_ENABLED`) | `true` | Auto-compress on threshold |
| Compression | `compression.threshold` | cfg+env (`CONTEXT_COMPRESSION_THRESHOLD`) | `0.85` | Fraction of context limit |
| Compression | `compression.summary_model` | cfg | `google/gemini-3-flash-preview` | Summarizer model |
| Terminal | `terminal.backend` | cfg+env (`TERMINAL_ENV`) | `local` | `local`/`docker`/`singularity`/`modal`/`ssh`/`daytona` |
| Terminal | `terminal.cwd` | cfg+env (`TERMINAL_CWD`, **deprecated env**) | per-backend | Working dir |
| Terminal | `TERMINAL_TIMEOUT` | env | `60` | Per-command timeout (s) |
| Terminal | `TERMINAL_LIFETIME_SECONDS` | env | `300` | Inactive-env cleanup |
| Terminal | `TERMINAL_DOCKER_IMAGE` | env | `nikolaik/python-nodejs:python3.11-nodejs20` | Docker image |
| Terminal | `TERMINAL_SINGULARITY_IMAGE` | env | `docker://nikolaik/python-nodejs:python3.11-nodejs20` | Singularity image |
| Terminal | `TERMINAL_MODAL_IMAGE` | env | `nikolaik/python-nodejs:python3.11-nodejs20` | Modal image |
| Terminal | `HERMES_DOCKER_BINARY` | env | `docker` | e.g. set to podman |
| Terminal | SSH | `TERMINAL_SSH_HOST/USER/PORT/KEY` | env | (unset) | Required when `terminal.backend=ssh` |
| Sudo | `SUDO_PASSWORD` | env | (unset) | Plaintext-on-disk; warned in `.env.example` (PH) |
| Browser | `BROWSERBASE_PROXIES` | env | `true` | Residential proxies |
| Browser | `BROWSERBASE_ADVANCED_STEALTH` | env | `false` | Scale-Plan stealth |
| Browser | `BROWSER_SESSION_TIMEOUT` | env | `300` | s |
| Browser | `BROWSER_INACTIVITY_TIMEOUT` | env | `120` | s |
| Memory | `memory.provider` | cfg | (per-setup) | `honcho` / `mem0` / `supermemory` / `byterover` / `hindsight` / `holographic` / `openviking` / `retaindb` |
| STT | `stt.provider` | cfg | `local` | priority `local > groq > openai` |
| STT | `stt.model` | cfg+env (`STT_GROQ_MODEL`, `STT_OPENAI_MODEL`) | per-provider | model id |
| Authz | `GATEWAY_ALLOW_ALL_USERS` | env | `false` | **single env trapdoor for open access (PH)** |
| Authz | `TEAMS_ALLOW_ALL_USERS` | env | `false` | Same for Teams |
| Authz | `TELEGRAM_ALLOWED_USERS` | env | (unset) | Comma-separated allowlist |
| Authz | `SLACK_ALLOWED_USERS` | env | (unset) | Comma-separated allowlist |
| Authz | `WHATSAPP_ALLOWED_USERS` | env | (unset) | Comma-separated allowlist |
| Authz | `EMAIL_ALLOWED_USERS` | env | (unset) | Comma-separated allowlist |
| Authz | `TEAMS_ALLOWED_USERS` | env | (unset) | AAD object IDs / UPNs |
| Pacing | `HERMES_HUMAN_DELAY_MODE` | env | `off` | `off`/`natural`/`custom` |
| Pacing | `HERMES_HUMAN_DELAY_MIN_MS` | env | `800` | custom-mode min |
| Pacing | `HERMES_HUMAN_DELAY_MAX_MS` | env | `2500` | custom-mode max |
| Telegram | `TELEGRAM_WEBHOOK_URL` | env | (unset) | Switches long-poll -> webhook mode |
| Telegram | `TELEGRAM_WEBHOOK_PORT` | env | `8443` | Webhook listen port |
| Telegram | `TELEGRAM_WEBHOOK_SECRET` | env | (unset) | Recommended in production |
| Telegram | `TELEGRAM_HOME_CHANNEL` (+ `_NAME`) | env | (unset) | Default chat for cron delivery |
| Email | `EMAIL_*` (ADDRESS, PASSWORD, IMAP_HOST/PORT, SMTP_HOST/PORT, POLL_INTERVAL=15) | env | (unset / 15s) | IMAP+SMTP credentials |
| Teams | `TEAMS_PORT` | env | `3978` | Bot Framework webhook port |
| Teams | `TEAMS_HOME_CHANNEL` (+ `_NAME`) | env | (unset) | Default chat |
| RL | `RL_API_URL` | env | `http://localhost:8080` | RL server endpoint |
| Skills hub | `GITHUB_TOKEN` | env | (unset) | Higher rate limits |
| Skills hub | `GITHUB_APP_ID` / `GITHUB_APP_PRIVATE_KEY_PATH` / `GITHUB_APP_INSTALLATION_ID` | env | (unset) | GitHub App identity for PRs |
| Voice | `VOICE_TOOLS_OPENAI_KEY` | env | (unset) | Whisper STT + OpenAI TTS |
| Voice | `GROQ_API_KEY` | env | (unset) | Free-tier Whisper |
| Debug | `WEB_TOOLS_DEBUG`, `VISION_TOOLS_DEBUG`, `MOA_TOOLS_DEBUG`, `IMAGE_TOOLS_DEBUG` | env | `false` | Per-toolset debug |
| Sessions | `save_trajectories` | cfg | `false` | Toggle JSON trajectory write |
| Skills | `skills.config.<key>` | cfg | per-skill | Each skill's frontmatter `metadata.hermes.config` keys |
| LLM provider keys | `OPENROUTER_API_KEY`, `GOOGLE_API_KEY`/`GEMINI_API_KEY`, `OLLAMA_API_KEY`, `GLM_API_KEY`, `KIMI_API_KEY`, `KIMI_CN_API_KEY`, `ARCEEAI_API_KEY`, `MINIMAX_API_KEY`/`MINIMAX_CN_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENCODE_GO_API_KEY`, `HF_TOKEN`, `XIAOMI_API_KEY` | env | (unset) | Each gates the corresponding provider adapter |
| LLM base URLs | `*_BASE_URL` overrides | env | per-vendor | E.g. `KIMI_BASE_URL=https://api.kimi.com/coding/v1` |
| Tool keys | `EXA_API_KEY`, `PARALLEL_API_KEY`, `FIRECRAWL_API_KEY`, `FAL_KEY`, `HONCHO_API_KEY`, `BROWSERBASE_API_KEY`, `BROWSERBASE_PROJECT_ID`, `TINKER_API_KEY`, `WANDB_API_KEY` | env | (unset) | Each gates the matching tool's `check_requirements()` |

*(observed fact — AGENTS.md "Adding Configuration" + .env.example exhaustively walked)*

### Defaults summary (per setting)

Most defaults are inherited from `DEFAULT_CONFIG` in `hermes_cli/config.py` (not directly read in this phase's budget — the canonical values are surfaced through AGENTS.md examples and `.env.example` placeholders). The values shown above for `display.skin=default`, `compression.enabled=true`, `compression.threshold=0.85`, `compression.summary_model=google/gemini-3-flash-preview`, `terminal.backend=local`, `stt.provider=local`, allowlist defaults `false`, pacing `off`/`800`/`2500`, terminal timeouts `60`/`300`, browser timeouts `300`/`120`, debug flags `false`, `RL_API_URL=http://localhost:8080`, `TEAMS_PORT=3978`, `EMAIL_POLL_INTERVAL=15` are all observed in those two sources.

*Open question:* exact `DEFAULT_CONFIG` literal not read in this phase — defer to porting phase if a single canonical doc is needed.

### Validation behavior

- Deep-merge handles additive changes automatically (no migration required for new keys).
- `_config_version` bumped only for active migration (key rename / structural change). Migration logic lives next to `load_config()`.
- `hermes config set` validates type/value at write-time (specific validator not directly read).
- Setup-wizard prompts use `OPTIONAL_ENV_VARS` metadata (`description`/`prompt`/`url`/`password`/`category`).
- Tools self-validate via `check_requirements()` returning bool; missing-env tools are hidden from `tools.registry` schemas.
