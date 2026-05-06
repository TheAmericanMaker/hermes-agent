# Mechanical Defects Report — hermes-agent

<!--
  Output template for the `defect-scan-mechanical` phase.
  Covers passes 1, 2, and 6 from the defect-scan methodology — the bugs visible
  from local code reading without contracts or protocols context.
  See findings/defect-scan-mechanical/SKILL.md for instructions.
-->

## Scan Context

- **Source:** `../` (repository root, hermes-agent at `claude/codecarto-hermes-analysis-abvQm`)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** `pipeline-full-with-deep-audit.yaml`
- **Date:** 2026-05-06
- **Scope:** Mechanical passes only (1 logic, 2 error handling, 6 configuration). Semantic passes (3 concurrency, 4 security, 5 contract violations) deferred to `defect-scan-semantic` after protocols.
- **Carry-forward inputs resolved:** arch-CF5 (`_last_resolved_tool_names` global — Pass 1), arch-CF6 (plugin-discovery import side effect — Pass 1 + Pass 6), arch-CF7 (ANSI escape leak — Pass 6).
- **Files read on hermes-agent source (within 30-file budget):** `model_tools.py`, `agent/retry_utils.py`, `agent/error_classifier.py`, `tools/environments/local.py`, `hermes_cli/config.py` (sampled), `gateway/config.py` (sampled), `pyproject.toml`, `.env.example`, plus directory listings of `agent/` and `tools/environments/`. The two configuration loaders were sampled rather than read end-to-end because of file-size limits; line citations below reference the sampled regions.

---

## Pass 1: Logic and Correctness

### Sorted findings (critical → low)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D1.1 | `model_tools.py:`*plugin-fallback path* (`_last_resolved_tool_names` reads in `handle_function_call` for `execute_code`) | `_last_resolved_tool_names` is a **process-global mutable list** rebuilt by every `get_tool_definitions(quiet_mode=True)` call (memoized cache hit *and* miss paths). Subagent calls (`delegate_tool._run_single_child`) and the gateway worker threads share the same global; if a subagent rebuilds the tool set with a narrower toolset, then yields, then the parent's next `execute_code` dispatch reads the global, the parent's sandbox gets the child's tool list. The `enabled_tools` parameter on `handle_function_call` was added to mitigate this, but the fallback to the global remains and is the *default* path for every caller that doesn't pass `enabled_tools` (CLI, gateway runner) — see `if enabled_tools is not None else _last_resolved_tool_names`. | high | observed fact | port differently |
| D1.2 | `model_tools.py` (module-import side effect — `discover_builtin_tools()` and `discover_plugins()` at lines following the "Tool Discovery" comment) | `discover_plugins()` runs at import time, wrapped in a bare `except Exception as e: logger.debug(...)`. A plugin that fails on import is *silently dropped* from the registry at DEBUG level — the user's tools just disappear with no visible warning unless DEBUG logging is enabled. The CLI's default log level is INFO. | high | observed fact | fix before porting |
| D1.3 | `agent/error_classifier.py:_extract_status_code` (depth-5 cause-chain walk) | `current = cause` after the `if cause is None or cause is current: break` check — but the loop's `for _ in range(5)` cap silently truncates legitimate longer cause chains. Five hops is more than nesting the project itself produces, but a chain seeded by an SDK that re-wraps via `raise X from Y from Z from W from V from U` would lose the originating HTTP status. Latent dead-end, not an active bug. | low | strong inference | leave behind |
| D1.4 | `agent/error_classifier.py:classify_api_error` (the SSL-vs-context disambiguation guard) | `_SSL_TRANSIENT_PATTERNS` includes both space-separated and underscore-separated forms (`"bad record mac"` and `"bad_record_mac"`). The `if any(p in error_msg for p in _SERVER_DISCONNECT_PATTERNS)` check at the next stage uses `"connection reset by peer"` — but on `RemoteProtocolError("Server disconnected without sending a response")` paired with a TLS alert, the SSL guard runs first and returns `timeout`, never exercising the disconnect-on-large-session compression path. This is the documented intent (comments say "SSL alerts mid-stream are transport hiccups, not server-side context overflow signals"), so functioning as designed — but if a *real* context-overflow disconnect ever rides on a TLS alert string (some proxies do this), it will be misclassified as transient. Logged as open question, not a hard bug. | low | open question | leave behind |
| D1.5 | `model_tools.py:_run_async` (timeout cancellation) | The fallback cancel-on-timeout path uses `worker_loop.call_soon_threadsafe(t.cancel)` for each pending task, then `pool.shutdown(wait=False)`. If `loop_ready.wait(timeout=1.0)` times out (no loop ever became ready — extremely tight race in worker startup), the cancel is silently skipped (the surrounding `if loop_ready.wait(timeout=1.0) and worker_loop is not None` short-circuits) and the worker thread leaks. Comment acknowledges "no-op on a running worker" trade-off, but the silent skip when the loop never reports ready is undocumented. | medium | observed fact | port differently |
| D1.6 | `tools/environments/local.py:_resolve_shell_init_files` | When `auto_bashrc` is True and explicit list is empty, candidates are `["~/.profile", "~/.bash_profile", "~/.bashrc"]` — `os.path.expandvars(os.path.expanduser(raw))` ignores `${HOME}`-style references that evaluate to a different value than `~`. Per the function's own docstring, this is intentional (`~` already expanded via `expanduser`). Not a defect — confirming clean. | n/a | observed fact | n/a |
| D1.7 | `agent/retry_utils.py:jittered_backoff` | When `attempt <= 0` (caller passes 0 or negative), `exponent = max(0, attempt - 1)` becomes 0 and `delay = base_delay`. That's the documented contract. `attempt == 1` correctly yields `base_delay`. Confirmed clean — flagged here only because the function is a building block for many retry paths and has no input validation. | n/a | observed fact | n/a |

### Carry-forward resolution in Pass 1

- **arch-CF5** (`_last_resolved_tool_names` global): resolved as **D1.1**. The architecture map's "process-global mutable list" hazard is a real cross-call shared mutable state. Mitigated by the `enabled_tools` parameter on `handle_function_call` (added later) but not eliminated — the fallback path still reads the global and the cache-hit path in `get_tool_definitions` *also* writes to it (not just the compute path). This is `port differently`: the port should pass tool sets through the call chain rather than relying on a process-global.
- **arch-CF6** (plugin discovery import side effect, logic angle): resolved as **D1.2**. The bare `except Exception` at module-level swallows plugin import errors at DEBUG level. The "import side effect" architecture concern manifests as a visible logic defect: a broken plugin disappears silently for the rest of the process lifetime.

---

## Pass 2: Error Handling and Resilience

### Sorted findings (critical → low)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D2.1 | `gateway/config.py:load_gateway_config` (the giant `try:` from ~line 622 wrapping all per-platform YAML→`os.environ` mappings, with single `except Exception as e` at line 928) | A **single broad `except`** wraps the entire 300-line YAML-to-env mapping for *every* platform (Slack, Discord, Telegram, WhatsApp, DingTalk, Matrix, Feishu). If parsing the Slack section raises (e.g. a malformed `slack.allow_mentions` map), every platform that comes after Slack is silently skipped and reverts to .env-only values. The `logger.warning` says "falling back to .env / gateway.json values" but the user never sees *which* platform's settings were dropped. | critical | observed fact | fix before porting |
| D2.2 | `model_tools.py:handle_function_call` — three back-to-back `try: ... except Exception: pass` blocks for plugin hooks (`get_pre_tool_call_block_message`, `notify_other_tool_call`, `invoke_hook("post_tool_call", ...)`, `invoke_hook("transform_tool_result", ...)`) | Plugin hook failures are silently swallowed with `pass` — no log line, no metric, no fail-open mode toggle. A plugin author's bug or a hook that itself raises in a thread-safety case will produce zero diagnostic. The comment "Fail-open" at `transform_tool_result` is the documented contract, but the absence of any log makes plugin debugging far harder than necessary. | high | observed fact | port differently |
| D2.3 | `model_tools.py:handle_function_call` (the outermost try/except wrapping `registry.dispatch`) | The outer `except Exception as e` returns `json.dumps({"error": error_msg}, ensure_ascii=False)`. Good — the LLM gets a structured error. **However:** `post_tool_call` and `transform_tool_result` hooks **are not called** when dispatch raises, because the hook calls live inside the same `try` block after the `result = ...` assignment. Plugin observers built around `post_tool_call` for latency dashboards/regression canaries will never see the error case. | high | observed fact | fix before porting |
| D2.4 | `tools/environments/local.py:_sanitize_subprocess_env` (and `_make_run_env`) | Imports of `tools.env_passthrough` are wrapped in `except Exception: _is_passthrough = lambda _: False`. If the `env_passthrough` module is broken, *every* env var is treated as non-passthrough — the user-configured passthrough list is silently dropped. No log, no warning. This is the same swallow-pattern as D2.2 but on a security-relevant code path (env var leakage to subprocesses). | high | observed fact | fix before porting |
| D2.5 | `tools/environments/local.py:LocalEnvironment.cleanup` | Both unlink calls are wrapped in `except OSError: pass`. The intent is "best effort cleanup on shutdown" but the same pattern would silently drop a real "device full" or "permission denied" state — leaking temp files across process restarts. Standard pattern, low-impact, but repeats across multiple environments. | low | observed fact | leave behind |
| D2.6 | `agent/retry_utils.py:jittered_backoff` | No upstream cap on `attempt` from callers' side: function caps at `max_delay` once `exponent >= 63`, but the seed math `(time.time_ns() ^ (tick * 0x9E3779B9)) & 0xFFFFFFFF` will work with any attempt count. Not a bug — just defense-in-depth. **Confirming clean** for this pass. | n/a | observed fact | n/a |
| D2.7 | `model_tools.py` MCP discovery removal (comment block) | The comment documents that MCP discovery used to block the gateway event loop for ~120s on first message and was moved out of import-time to per-entrypoint. The new entrypoints (gateway, cli, hermes_cli, tui_gateway, acp_adapter) each "run discovery explicitly at its own startup." This means MCP discovery is duplicated across N entrypoints with N independent timeout policies; if one entrypoint adds a new init path and forgets to call discovery, MCP tools simply don't show up — silent feature absence. | medium | strong inference | port differently |
| D2.8 | `agent/error_classifier.py:_extract_error_body` (response body fetch) | `response.json()` is wrapped in `except Exception: pass` and an empty `{}` is returned. Patterns that depend on body parsing (`_classify_400` generic-with-large-session logic) silently lose information when the body is HTML, malformed JSON, or compressed without proper decode. The classifier returns `unknown` retryable=True, so the loop retries — masking a non-retryable backend error as transient. | medium | strong inference | port differently |

### Carry-forward resolution in Pass 2

- **arch-CF6** (import-order angle): the plugin-discovery side effect *also* shows up here as **D2.7** — moving discovery out of import-time (correct architectural fix) was implemented by duplicating the discovery call across every entrypoint, each with its own implicit "is the loop running yet?" assumption. This is the trade-off: the original blocking-import behaviour was worse, but the replacement is fragile to entrypoint drift.

---

## Pass 6: Configuration and Environment Hazards

### Sorted findings (critical → low)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D6.1 | `gateway/config.py:load_gateway_config` (~lines 750-926: dozens of `os.environ[...] = ...` assignments) | Loading config.yaml **mutates `os.environ`** by writing dozens of platform-specific env vars (`SLACK_REQUIRE_MENTION`, `DISCORD_AUTO_THREAD`, `TELEGRAM_PROXY`, `WHATSAPP_DM_POLICY`, etc.). Side-effect: any subsequent code path that reads `os.getenv` after gateway init sees values that the user did not set in their `.env`. This breaks the documented precedence ("env vars take precedence over config.yaml") because by the time `_apply_env_overrides` runs, the env vars *have* been set from yaml. The guard `not os.getenv(KEY)` mostly works, but `getenv` returns `None` only for unset, not for empty-string — a `.env` line `SLACK_ALLOW_BOTS=` (empty) is treated as set, while a yaml `allow_bots: false` is dropped. | critical | observed fact | fix before porting |
| D6.2 | Multiple config loaders — `hermes_cli.config.load_config()`, `hermes_cli.config.read_raw_config()`, `gateway.config.load_gateway_config()`, plus direct `yaml.safe_load(f)` reads inside `gateway/config.py` and (per AGENTS.md) inside other files. | Three+ separate config loaders for the *same* `~/.hermes/config.yaml` file, each with its own caching strategy: `_LOAD_CONFIG_CACHE` in `hermes_cli/config.py` is mtime-and-size keyed; `_RAW_CONFIG_CACHE` is a separate cache for the raw form; `gateway/config.py` reads the file inline with no caching. Cache invalidation is by file mtime in two places and absent in the third — a config change between the gateway loader's read and an `hermes_cli` call may produce inconsistent views in the same process. | high | observed fact | port differently |
| D6.3 | `hermes_cli/config.py:_secure_dir` (line 257) | `_secure_dir` chmods `~/.hermes/` to 0o700 by default unless `HERMES_HOME_MODE` is set. If the user sets `HERMES_HOME_MODE=invalid_octal`, `int(mode_str, 8)` raises `ValueError` and the function silently falls back to `0o700` (good). However, if the user sets `HERMES_HOME_MODE=0777`, the directory is chmod'd 0777 — world-writable secrets directory with no warning. There is no minimum-restrictiveness check. | high | observed fact | fix before porting |
| D6.4 | `hermes_cli/config.py:_expand_env_vars` (line 3583) | `${VAR}` references in config.yaml that don't resolve in `os.environ` are silently kept as the literal `${VAR}` string. A typo'd env var name (`${OPENROOTER_API_KEY}`) becomes a literal API key string that fails opaquely at provider call time. Documented as intentional ("Unresolved references are kept verbatim so callers can detect them") but no caller is observed to do that detection — a `${...}` literal will reach `httpx` as the auth token. | high | observed fact | fix before porting |
| D6.5 | `tools/environments/local.py:_HERMES_PROVIDER_ENV_BLOCKLIST` (large hardcoded list of provider env-var names) | Hardcoded blocklist of provider/platform env vars to strip from subprocess environments. Each new provider added must remember to add its `*_API_KEY` here. Latent leak risk: the blocklist diverges from the actual provider registry over time. Mitigated by `PROVIDER_REGISTRY` and `OPTIONAL_ENV_VARS`-based dynamic injection at the top of `_build_provider_env_blocklist`, but the hardcoded fallback (lines listing OPENAI_BASE_URL, ANTHROPIC_TOKEN, etc.) remains the canonical source. | medium | observed fact | port differently |
| D6.6 | ANSI escape leak (arch-CF7) — `agent/display.py` known-pitfall referenced in AGENTS.md, not directly read in this scan | AGENTS.md flags the documented pitfall: ANSI `\033[K` escapes from `KawaiiSpinner` / `prompt_toolkit.patch_stdout` can leak as literal text into transcripts, gateway streams, and `logs/agent.log`. Verifying this requires reading `agent/display.py` (38 KB), which was deferred under the read budget. Routed as **open question** for the contracts/protocols phase to verify by examining the actual log files / gateway message envelopes, since it is observable behaviour rather than a static-readable defect. | medium | open question | port differently |
| D6.7 | `gateway/config.py` int-conversions of env vars — e.g. `int(os.getenv("WECOM_CALLBACK_PORT", "8645"))`, `int(os.getenv("BLUEBUBBLES_WEBHOOK_PORT", "8645"))` | No try/except around `int()`. If the user sets `WECOM_CALLBACK_PORT=abc`, gateway init crashes with an unhelpful `ValueError`. Also: same default port `8645` used for both `WECOM_CALLBACK_PORT` and `BLUEBUBBLES_WEBHOOK_PORT` — running both adapters at once in default config will fail with bind-address-already-in-use, and the error surface is the second adapter to start, not the config layer. | high | observed fact | fix before porting |
| D6.8 | `model_tools.py` plugin discovery import side effect (arch-CF6, env angle) | `discover_plugins()` runs at module import. Plugins may write to disk, open network connections, register CLI commands, or read environment variables. Importing `model_tools` thus has a non-trivial startup cost that is not visible to users of the module. The architecture-map flagged this; from a config-and-env hazard perspective, the side effect makes "what env vars does importing module X consult?" unanswerable without running the imports. | medium | observed fact | port differently |
| D6.9 | `.env.example` lines `TERMINAL_TIMEOUT=60`, `TERMINAL_LIFETIME_SECONDS=300`, `BROWSER_INACTIVITY_TIMEOUT=120` | Default values are baked into `.env.example` as positional defaults *and* into `hermes_cli/config.py` `DEFAULT_CONFIG` (line 391+). If a user copies `.env.example` to `.env` they pick up these specific values — but config.yaml's `DEFAULT_CONFIG` may have different defaults (e.g. `agent.gateway_timeout: 1800`). Drift between the two default sources is a source of subtle behavioural differences for fresh installs. | medium | observed fact | port differently |
| D6.10 | `tools/environments/local.py:get_temp_dir` | Falls back to `/tmp` even when neither `/tmp` exists (Termux) nor `tempfile.gettempdir()` returns a POSIX path. Final hard-coded fallback: `return "/tmp"`. On Windows or constrained sandboxes where neither path exists, the resulting subprocess will fail with no-such-file errors that don't point at `get_temp_dir` as the source. | low | observed fact | port differently |

### Carry-forward resolution in Pass 6

- **arch-CF7** (ANSI escape leak): partially addressed as **D6.6** — the AGENTS.md known-pitfall list confirms the leak exists; the scan deferred direct verification (would require reading `agent/display.py` and a sample of `agent.log`/`gateway.log`). Logged as open question.
- **arch-CF6** (plugin-discovery startup cost): resolved as **D6.8**.

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 2 |
| High | 8 |
| Medium | 7 |
| Low | 4 |
| **Total** | 21 |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 1. Logic and correctness | 0 | 2 | 1 | 2 | 5 (+ 2 confirmed-clean entries) |
| 2. Error handling | 1 | 3 | 2 | 1 | 7 (+ 1 confirmed-clean entry) |
| 6. Config and environment | 1 | 3 | 4 | 1 | 9 |

(Counts include entries that contain a real finding; confirmed-clean rows D1.6, D1.7, D2.6 are excluded from the severity totals.)

### Top Findings

1. **D6.1** (Pass 6, critical) — `gateway/config.py` mutates `os.environ` from config.yaml during boot, breaking the documented "env vars take precedence" precedence.
2. **D2.1** (Pass 2, critical) — `gateway/config.py` wraps all per-platform YAML mapping in one broad `except Exception`, silently dropping every platform after the first failure.
3. **D6.4** (Pass 6, high) — `_expand_env_vars` keeps unresolved `${VAR}` literals; typo'd env var names reach `httpx` as auth tokens.
4. **D6.3** (Pass 6, high) — `_secure_dir` accepts any `HERMES_HOME_MODE` value (including `0777`) with no minimum-restrictiveness check.
5. **D1.1** (Pass 1, high) — `_last_resolved_tool_names` process-global creates subagent-vs-parent crosstalk on the `execute_code` sandbox tool list.
6. **D1.2** (Pass 1, high) — plugin discovery failures silently logged at DEBUG; broken plugins disappear with no user-visible signal.
7. **D6.7** (Pass 6, high) — unguarded `int(os.getenv(...))` in gateway config + duplicate default port 8645 for two adapters.
8. **D2.4** (Pass 2, high) — `env_passthrough` import failure silently disables passthrough, dropping user config on a security-relevant path.

### Routed To Semantic Phase

The following items were spotted during this scan but are concurrency, security, or contract-drift in nature, and are routed to `defect-scan-semantic`. These will appear under `phases.defect-scan-mechanical.carry_forward` in `status.yaml` (orchestrator-managed):

| ID | Description | Why Routed |
|----|-------------|-----------|
| dsm-CF1 | `_last_resolved_tool_names` race between subagent ThreadPoolExecutor workers and the parent agent thread (`model_tools.py`). D1.1 documents the logic shape; the actual TOCTOU window between worker write and parent read is a concurrency contract. | Concurrency / pass 3 territory. |
| dsm-CF2 | `_jitter_counter` in `agent/retry_utils.py` is locked, but seed-derivation timing (`time.time_ns()` mixed with counter) means two concurrent retries from different sessions can still produce correlated jitter for the *same* `attempt` value when clocks are coarse — a soft thundering-herd risk despite the documented anti-correlation. | Concurrency / pass 3. |
| dsm-CF3 | `gateway/config.py` writes provider/platform tokens into `os.environ` at gateway startup (e.g. `SLACK_FREE_RESPONSE_CHANNELS`, `TELEGRAM_PROXY`). Subprocesses spawned by terminal backends inherit these unless `_HERMES_PROVIDER_ENV_BLOCKLIST` covers each, but the blocklist is hardcoded and doesn't include `*_FREE_RESPONSE_CHANNELS` or `*_PROXY`. Subprocess token leakage. | Security / pass 4. |
| dsm-CF4 | `agent/error_classifier.py` retries on `unknown` with `retryable=True`. A 404 to a misconfigured local llama.cpp endpoint loops indefinitely against a non-recoverable misconfig — DoS-amplifies to the upstream provider. Combined with `_classify_400` falling through to `format_error` only when `is_generic and is_large` is False, the retry ceiling is enforced upstream by `agent.max_turns` / `api_max_retries=3`. Investigate the contract: should `unknown` after N retries demote to non-retryable? | Contract / pass 5. |
| dsm-CF5 | `model_tools.py:handle_function_call` runs `post_tool_call` and `transform_tool_result` hooks **only on success** — D2.3. Plugin contract documented in architecture says hooks should observe every dispatch, including failures. API-contract violation. | Contract / pass 5. |
| dsm-CF6 | The `transform_tool_result` hook silently picks the first plugin's string return ("first valid string return wins") — order-dependent contract that depends on plugin discovery order, which depends on filesystem iteration order. Non-deterministic across deploys. | Contract / pass 5. |

---

## Validation

(See validation block, appended in a separate commit per VALIDATE.md.)
