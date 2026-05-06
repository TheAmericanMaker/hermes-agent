# Runtime Lifecycle

## 2026-05-06 — architecture phase

All entries `observed fact` from AGENTS.md, README.md, and direct file/dir listings unless marked.

### Boot sequence — `hermes` interactive CLI

1. `hermes_cli/main.py` argparse parses top-level flags (`-p <profile>`, `--tui`, subcommand).
2. `_apply_profile_override()` sets `HERMES_HOME` env var **before any other module imports** so that `get_hermes_home()` resolves to the correct profile dir.
3. `hermes_cli/env_loader.py` loads `~/.hermes/.env` via `python-dotenv`.
4. Subcommand dispatch — if interactive REPL: instantiate `cli.HermesCLI`.
5. `HermesCLI` calls `load_cli_config()` (merges hardcoded defaults + user YAML).
6. `init_skin_from_config()` reads `display.skin` and caches active `SkinConfig`.
7. `agent/skill_commands.py` scans `~/.hermes/skills/` and registers slash commands.
8. `discover_plugins()` runs as a side effect of importing `model_tools.py` (or is called explicitly).
9. `discover_builtin_tools()` triggers eager-import of every `tools/*.py`; each module calls `tools.registry.register()` at import time.
10. Banner rendered (`hermes_cli/banner.py`); welcome message from skin's `branding.welcome`.
11. `AIAgent` instantiated lazily on first user turn (or eagerly per config). Constructor takes ~60 params: credentials, routing, callbacks, session context, budget, credential_pool, prefill_messages, service_tier, reasoning_config, etc.
12. Memory manager + context engine + curator initialized (subject to `skip_memory`).
13. Context files loaded unless `skip_context_files`.
14. System prompt assembled by `agent/prompt_builder.py` with cache-stable layout.
15. Enter interactive prompt via `prompt_toolkit` with `SlashCommandCompleter`.

### Boot sequence — Gateway (`hermes gateway start` or `hermes-agent`)

1. Steps 1–5 above.
2. `gateway/run.py` reads `~/.hermes/config.yaml` raw via `gateway/config.py` (a third config loader, distinct from `load_cli_config()` and `load_config()`).
3. `gateway/platform_registry.py` enumerates enabled platforms based on env vars and config.
4. For each platform: instantiate adapter (subclass of `gateway/platforms/base.py:BasePlatformAdapter`), call `acquire_scoped_lock()` from `gateway.status` (token-scoped, prevents two profiles from sharing the same bot credential), call `adapter.connect()`/`adapter.start()`.
5. `gateway/stream_consumer.py` opens shared output stream consumer.
6. Cron scheduler started (`cron/scheduler.py` + `cron/jobs.py`); jobs delivered via `gateway/delivery.py`.
7. Per-conversation `GatewaySession` (`gateway/session.py`, ~56KB) instances created on demand; each owns an `AIAgent`.

### Boot sequence — TUI (`hermes --tui`)

1. Node.js Ink process launched (entry `ui-tui/src/entry.tsx`).
2. Ink spawns Python `tui_gateway/server.py` over stdio with newline-delimited JSON-RPC.
3. Python server initializes `AIAgent` and emits `gateway.ready` event with skin data.
4. Ink subscribes to event stream; renders transcript, composer, prompts, activity feed.

### Boot sequence — Dashboard (`hermes dashboard`)

1. FastAPI/uvicorn started by `hermes_cli/web_server.py` (binds 127.0.0.1 by default).
2. Browser hits `/`; React SPA from `web/dist` (packaged into `hermes_cli/web_dist/`) served.
3. Ephemeral `_SESSION_TOKEN` issued; SPA opens `/api/pty?token=...` WebSocket.
4. Server spawns `hermes --tui` over a PTY (`ptyprocess`); raw bytes proxied each direction.
5. `\x1b[RESIZE:cols;rows]` frames intercepted server-side and translated to `TIOCSWINSZ` ioctl.

### Main loop — Agent (`run_conversation()`)

Synchronous (blocking) `while` loop:

```
while (api_call_count < max_iterations and iteration_budget.remaining > 0) or _budget_grace_call:
    if _interrupt_requested: break
    response = client.chat.completions.create(model, messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            (pre_tool_call hooks)
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            (post_tool_call hooks)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

- Default `max_iterations`: 90.
- Iteration anatomy: provider call → tool dispatch (sequential, not parallel) → tool result append → budget check → interrupt check.
- Reasoning content stored in `assistant_msg["reasoning"]`.
- Pre/post LLM call hooks run in `run_agent.py`; pre/post tool hooks run in `model_tools.py`.
- Context compression triggered by `compression.threshold` (default 0.85) when conversation approaches model's context limit. **Only legitimate mid-conversation system-prompt mutation point** — prompt-cache invariant otherwise.

### Main loop — Gateway

- Each platform adapter runs its own asyncio event loop / consumer.
- Incoming message guarded by `gateway/platforms/base.py` `_pending_messages` queue if `session_key in self._active_sessions`.
- Then guarded again by `gateway/run.py` command interceptor for `/stop`, `/new`, `/queue`, `/status`, `/approve`, `/deny`.
- Otherwise dispatched to `_process_message_background()` which acquires/creates a `GatewaySession`.
- Background process watcher (when `terminal(background=true, notify_on_complete=true)`) detects process completion and triggers a new agent turn.

### Main loop — TUI

- Slash worker subprocess `_SlashWorker` is persistent; `slash.exec` requests dispatched there, with fallthrough to `command.dispatch`.
- Streaming responses delivered as `message.delta` events; finalized with `message.complete`.
- Tool activity reported as `tool.start` / `tool.progress` / `tool.complete`.
- Approval requests surfaced via `approval.request` events; user response via `approval.respond`.

### Shutdown

- CLI: flushes session DB, closes trajectory file (if `save_trajectories`), tears down skin engine cache, returns from `main()`.
- Gateway: each adapter's `disconnect()`/`stop()` releases its scoped credential lock; cron scheduler stopped; in-flight `GatewaySession` instances allowed to drain.
- TUI: Node Ink exits; Python `tui_gateway` server gets EOF on stdin and shuts down.
- *Open question (arch-OQ1)*: precise ordering of credential-pool flush vs platform lock release vs session DB flush — not directly read.

### Background tasks

- Cron scheduler (`cron/`).
- Background terminal processes (configurable via `display.background_process_notifications`: `all`/`result`/`error`/`off`).
- Memory curator (`agent/curator.py`) runs periodic memory consolidation.
- Title generator (`agent/title_generator.py`) generates session titles asynchronously.
- Insights generator (`agent/insights.py`) builds usage analytics.

## 2026-05-06 — protocols phase

This section appends event-ordered detail derived from direct reads of `mcp_serve.py`, `acp_adapter/entry.py`, `gateway/session.py`, `gateway/stream_consumer.py`, `gateway/config.py` (Platform/SessionResetPolicy/HomeChannel), `gateway/platforms/telegram.py` (sample adapter), and grep-extracted method names from `tui_gateway/server.py`. Together they resolve part of arch-OQ1 (shutdown / drain) and ground the gateway listener lifecycle and agent-turn anatomy used in `findings/protocols/protocols-and-state.md` §State Machines.

### Full event-ordered boot sequence — gateway, with locks acquired

*(observed fact, except where marked)*

1. `hermes-agent` (or `hermes gateway start`) entry → `run_agent.main` → `gateway/run.py`.
2. `_apply_profile_override()` sets `HERMES_HOME`. *(observed fact)*
3. `load_hermes_dotenv(hermes_home=...)` loads `.env` from active profile. *(observed fact — `acp_adapter/entry.py` mirrors this)*
4. `gateway/config.GatewayConfig` constructed by reading `config.yaml` raw (third config loader, separate from `hermes_cli.config.load_config` and `hermes_cli.config.load_cli_config`).
5. `gateway/platform_registry.platform_registry` enumerates entries; for each, an adapter subclass of `BasePlatformAdapter` is instantiated.
6. Per adapter, in order: (a) `acquire_scoped_lock(token)` from `gateway.status` — token-scoped to prevent two profiles sharing the same bot credential; (b) adapter `connect()` / `start()`; (c) registers inbound message callback into `_pending_messages` + command-interceptor pipeline.
7. `SessionStore(sessions_dir, GatewayConfig, has_active_processes_fn)` is constructed; `_db = SessionDB()` opens `state.db` in WAL mode. *(observed fact — `gateway/session.py`)*
8. `SessionStore.suspend_recently_active(120)` is called on startup so any session that was active within 120s of the previous gateway exit is forced to fresh state on next access — prevents resuming sessions that may have been mid-tool-call. *(observed fact)*
9. Cron scheduler started (`cron.scheduler` + `cron.jobs`). 
10. Stream consumer factory ready (per-turn `GatewayStreamConsumer` instances created on first stream callback).
11. Steady-state: each platform listener pumps inbound messages.

### Agent turn anatomy (deep)

- Pre: `pre_llm_call` plugin hooks run.
- Provider call: blocking. Tenacity retries on transient errors. `credential_pool` rotates keys on quota errors. `nous_rate_guard` pre-check.
- On `response.tool_calls`: sequential dispatch (`for tool_call in response.tool_calls`) — NOT parallel even when provider supplied parallel calls. Each tool: `pre_tool_call` hook → `model_tools.handle_function_call(name, args, task_id)` → `post_tool_call` hook → `messages.append({"role":"tool","tool_call_id":..., "content": result_json_str})`.
- Post: when `response.tool_calls` empty, `post_llm_call` hooks run; final content returned.
- Stream callback: each token delta forwarded to `stream_delta_callback(text)` if registered; `text=None` is a tool-boundary sentinel consumed by `GatewayStreamConsumer` as `_NEW_SEGMENT`.
- Iteration budget: hard limit at `max_iterations` (default 90) AND soft limit `iteration_budget.remaining` with one-turn grace.
- Interrupt: checked every iteration via `_interrupt_requested` (set by gateway `/stop` interceptor).

### Gateway listener lifecycle (resolves arch-CF4)

States and transitions are enumerated in `findings/protocols/protocols-and-state.md` §State Machines #1. Highlights:

- **Lock acquisition is fail-fast**: `acquire_scoped_lock` blocks/rejects when another profile already holds the same bot token.
- **Drain timeout**: if a clean shutdown can't drain in-flight sessions, `mark_resume_pending(key, reason="restart_timeout")` is set so the next start preserves `session_id` and resumes the transcript intact (`clear_resume_pending` fires after the next successful turn).
- **Suspend wins over resume**: `/stop` (or stuck-loop escalation) sets `suspended=True`; this ALWAYS forces a fresh `session_id` on next access regardless of `resume_pending`.
- **Two-guard pattern** (portability hazard, routed to defect-scan-semantic prot-CF1): `_pending_messages` queue at adapter level + command interceptor at `gateway/run.py` for `/stop`,`/new`,`/queue`,`/status`,`/approve`,`/deny`,`/platforms`,`/sethome` — every new control command MUST bypass BOTH.

### Stream consumer lifecycle (per turn)

*(observed fact — `gateway/stream_consumer.py`)*

- Sentinels: `_DONE`, `_NEW_SEGMENT`, `_COMMENTARY`.
- `_message_id` may be `None` (no message yet) | a real id | `__no_edit__` (platform accepted the send but returned no editable id; subsequent text goes via `_send_fallback_final`).
- Adaptive flood backoff: `_current_edit_interval *= 2` cap 10s; after 3 strikes `_edit_supported = False` and consumer enters fallback.
- Fresh-final: when `fresh_final_after_seconds > 0` and a preview has been visible long enough, the final edit is replaced by a fresh send + best-effort delete of the old preview, so the platform timestamp reflects completion (port of openclaw#72038).
- `<think>` / `<reasoning>` / `<REASONING_SCRATCHPAD>` block filter: state machine with `outside_think` / `inside_think` / `partial_tag_held` (boundary-aware to avoid false positives in prose). Tag list duplicated in `cli.py`, `run_agent.py`, `gateway/stream_consumer.py` — portability hazard prot-CF6.

### Shutdown sequence (resolves part of arch-OQ1)

*(strong inference, since the exact code path inside 634KB `gateway/run.py` was not read end-to-end; corroborated by `gateway/session.py` `suspend_recently_active` and `mark_resume_pending` semantics)*

Probable order:

1. SIGTERM / KeyboardInterrupt received in `gateway/run.py` main loop.
2. Each platform adapter is asked to stop accepting new inbound (listener task cancel).
3. In-flight `GatewaySession` turns are awaited; on drain timeout, `SessionStore.mark_resume_pending(session_key, reason="restart_timeout")` per stuck session.
4. Cron scheduler stopped.
5. Each adapter's `disconnect()` releases its scoped credential lock via `gateway.status.release_scoped_lock`.
6. `SessionDB` writers flushed (WAL checkpoint implicit at process exit).
7. Process exits.

*(open question prot-OQ-shutdown-precise: exact ordering of credential-pool flush vs scoped-lock release vs DB checkpoint — defer to defect-scan-semantic if a defect surfaces.)*

### Restart / resume semantics

- Session-id stable across restart unless `suspended=True` (forces fresh) or `was_auto_reset=True` (idle/daily policy fired).
- `expiry_finalized=True` persisted to `sessions.json` so the background expiry watcher does not re-finalize after a restart.
- `is_fresh_reset=True` flag is consumed once by the gateway message handler to trigger topic/channel skill re-injection on the first message of an explicitly reset session.
- The MCP `EventBridge` resets its monotonic cursor to 0 each start (no cursor persistence) — clients must tolerate cursor restart.
