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
