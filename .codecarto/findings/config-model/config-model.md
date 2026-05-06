# Config Model

## 2026-05-06 — architecture phase

### Sources & precedence (low → high)

1. `DEFAULT_CONFIG` baked into `hermes_cli/config.py` (versioned via `_config_version`; deep-merged on load).
2. `~/.hermes/config.yaml` (user YAML; merged over defaults).
3. `~/.hermes/.env` (secrets only — API keys, tokens, passwords). Loaded by `hermes_cli/env_loader.py` via `python-dotenv`.
4. Process environment variables (env-only override surface; some bridged to/from config.yaml — e.g. `terminal.cwd` → `TERMINAL_CWD` for child tools).
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

- New `config.yaml` option → add to `DEFAULT_CONFIG`. Bump version only if migrating.
- New secret → add to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py` with metadata: `description`, `prompt`, `url`, `password` (bool), `category` (`provider`/`tool`/`messaging`/`setting`).
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
