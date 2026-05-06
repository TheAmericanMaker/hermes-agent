# Protocols and State

<!-- Phase 4 primary output. Validation block appended in a follow-up commit. -->

Evidence levels:
- **observed fact** — direct from code, schemas, or types in the source tree
- **strong inference** — protocol-level conclusion from multiple facts
- **portability hazard** — assumption tied to source language/runtime/SDK
- **open question** — missing or conflicting behavior

Architecture references: `findings/architecture/architecture-map.md`. Carry-forward resolution: arch-CF2 (RPC schemas), arch-CF3 (persistent schemas), arch-CF4 (gateway state machine).

## Boundaries Identified

1. **Process ↔ process**
   - `hermes --tui`: Node Ink ↔ Python `tui_gateway/server.py` over **stdio**, newline-delimited JSON-RPC.
   - `hermes-acp` / `hermes acp`: Editor (Zed/VS Code/JetBrains) ↔ `acp_adapter/entry.py` over **stdio**, ACP JSON-RPC.
   - `hermes mcp serve`: MCP client (Claude Desktop, Cursor, Codex) ↔ `mcp_serve.py` over **stdio**, FastMCP framing.
   - Dashboard PTY child: `hermes_cli/web_server.py` ↔ `hermes --tui` subprocess over **PTY** + custom resize escape.
   - Subagent / delegate: parent `AIAgent` ↔ `_run_single_child()` child agent in same process (in-process Queue pattern). *(observed fact — AGENTS.md `_last_resolved_tool_names`)*
2. **UI ↔ core**
   - Interactive CLI (`cli.py` `HermesCLI`) ↔ `AIAgent` via in-process callbacks (`stream_delta_callback`, `tool_callback`, `error_callback`).
   - Ink TUI ↔ Python: `gateway.*` / `session.*` / `tool.*` / `message.*` / `slash.*` / `command.*` / `approval.*` JSON-RPC methods + `event` notifications.
   - Dashboard SPA ↔ FastAPI `/api/pty` over **WebSocket** carrying raw PTY bytes + `\x1b[RESIZE:cols;rows]` control frames.
3. **Core ↔ provider** (LLM wire)
   - OpenAI-compatible Chat Completions: `client.chat.completions.create(model, messages, tools=tool_schemas)` — used by all providers via `auxiliary_client`. *(observed fact — runtime-lifecycle.md `run_conversation`)*
   - Anthropic Messages API via `agent/anthropic_adapter.py` (mapped into the same internal message shape).
   - AWS Bedrock invoke via `agent/bedrock_adapter.py`.
   - Codex Responses (`agent/codex_responses_adapter.py`), Gemini native + CloudCode adapters.
4. **Tool layer ↔ runtime**
   - LLM tool-call → `model_tools.handle_function_call(name, args, task_id)` → registered handler returns JSON string.
   - Pre/post tool-call hooks fire around handler.
   - Side-channel: `delegate_tool` spawns subagents that use the same registry but mutate `_last_resolved_tool_names` global. *(portability hazard)*
5. **Runtime ↔ persistence**
   - `hermes_state.SessionDB` (SQLite + WAL + FTS5) — sessions / messages / state_meta tables.
   - `gateway/sessions.json` — session-key → session-id index, atomic-replace writes.
   - `logs/<session_id>.jsonl` legacy transcript files (still written; preferred-by-length on read).
   - `kanban_db.py` (SQLite) — kanban tracker.
   - `aiosqlite` / `asyncpg` — used by gateway listener side; chosen via config (asyncpg only when `gateway` configured against Postgres). *(strong inference — pyproject extras + AGENTS.md)*
6. **Local files ↔ exported artifacts**
   - `cron/output/` for cron job results when `deliver="local"`.
   - Sticker cache, browser session state, Playwright browsers, MCP OAuth tokens, Honcho/Qwen creds.
   - Trajectory JSON files (`logs/session_*.json`) consumed by `trajectory_compressor.py` and `batch_runner.py`.

## Event Catalog

### MCP Server — `mcp_serve.py` (stdio FastMCP)
*(observed fact — `mcp_serve.py` reviewed end-to-end)*

| Field | Value |
|---|---|
| Producer | `FastMCP` runtime in `mcp_serve.run_mcp_server()` |
| Consumer | Any MCP client (Claude Desktop, Cursor, Codex) |
| Transport | stdio JSON-RPC, framed by FastMCP |
| Tools (10) | `conversations_list`, `conversation_get`, `messages_read`, `attachments_fetch`, `events_poll`, `events_wait`, `messages_send`, `permissions_list_open`, `permissions_respond`, `channels_list` |
| Required fields | Per-tool args (e.g. `session_key`, `target`, `message`, `id`+`decision`, `after_cursor`) |
| Optional fields | `platform` filter, `limit`, `search`, `timeout_ms` (≤300000ms cap) |
| Identifiers | `session_key` (gateway-built), `session_id`, `chat_id`, `message_id`, monotonic `cursor` int |
| Ordering guarantees | Cursor-monotonic; polling FIFO. Background poll interval 200ms. Queue capped at 1000 events (FIFO trim). |
| Error cases | Returns `{"error": ...}` JSON string in tool result; never raises. `permissions_respond` returns `{"error": "Approval not found"}` if id unknown. Decision must be one of `allow-once`/`allow-always`/`deny`. |
| Restart/resume | EventBridge resets cursor to 0 on each `run_mcp_server`. Stale approvals from prior runs are NOT replayed (live-session-only). `_last_poll_timestamps` re-populated by polling. *(portability hazard — clients lose pre-startup approvals)* |

### ACP Adapter — `acp_adapter/entry.py` + `acp_adapter/server.py`
*(observed fact — `acp_adapter/entry.py` reviewed)*

| Field | Value |
|---|---|
| Producer | `acp.run_agent(HermesACPAgent, use_unstable_protocol=True)` |
| Consumer | Editor IDE (Zed/VS Code/JetBrains) |
| Transport | stdio JSON-RPC; stdout reserved for ACP frames; stderr for logs |
| Required fields | ACP schema (per `acp` package); HermesACPAgent class methods provide handlers |
| Liveness probes | Methods `ping`, `health`, `healthcheck` are NOT in ACP schema; server returns JSON-RPC `-32601` (method_not_found); `_BenignProbeMethodFilter` suppresses the noisy traceback in stderr only for these names. *(observed fact — entry.py)* |
| Error cases | `RequestError code=-32601` returned to caller; non-probe methods still surface tracebacks |
| Restart/resume | MCP tool discovery (`discover_mcp_tools()`) runs once at startup before `asyncio.run()` to avoid blocking the event loop on lazy import (#16856). Per-session MCP servers registered dynamically inside the loop via `asyncio.to_thread`. |

### TUI Gateway — `tui_gateway/server.py` (stdio, newline-delimited JSON-RPC)
*(observed fact — method names enumerated by grep over server.py)*

| Field | Value |
|---|---|
| Producer / Consumer | Bidirectional: Ink Node process (TS) ↔ Python server (Python). |
| Transport | stdio newline-delimited JSON-RPC 2.0; envelope: `{"jsonrpc":"2.0","method":..., "params":...}` for events, `{"jsonrpc":"2.0","id":rid,"result":...}` for replies, `{"jsonrpc":"2.0","id":rid,"error":{"code":,"message":}}` for errors. |
| Methods (≥27) | `gateway.*` (ready, stderr); `session.create`, `session.list`, `session.info`, `session.history`, `session.most_recent`, `session.resume`, `session.save`, `session.delete`, `session.close`, `session.title`, `session.compress`, `session.steer`, `session.undo`, `session.branch`, `session.usage`, `session.interrupt`; `slash.exec`; `command.dispatch`, `command.resolve`; `tool.start`, `tool.started`, `tool.generating`, `tool.progress`, `tool.complete`; `message.start`, `message.delta`, `message.complete`; `approval.request`, `approval.respond`. |
| Required fields | Each method-specific (session_id, args dict). |
| Ordering guarantees | Per-session events are ordered (single-writer dispatch loop). Concurrency: long-running sessions can suspend the dispatcher loop in entry.py (#12546 noted); compaction yields. |
| Error cases | JSON-RPC error envelope; `gateway.stderr` event re-emits panic traces; `tui_gateway_crash.log` written by `_panic_hook` on unhandled exceptions. |
| Restart/resume | Ink can re-spawn the Python child; `session.resume` re-attaches a previous `session_id`. Crash log under `~/.hermes/logs/tui_gateway_crash.log`. |

### Dashboard PTY WebSocket — `hermes_cli/web_server.py` `/api/pty`
*(observed fact — runtime-lifecycle.md + AGENTS.md)*

| Field | Value |
|---|---|
| Producer | `hermes_cli/web_server.py` (FastAPI/uvicorn) |
| Consumer | Browser SPA from `web/dist` |
| Transport | WebSocket, binary frames (raw PTY stdout/stdin) plus inline ANSI escape `\x1b[RESIZE:<cols>;<rows>]` (consumed server-side, translated to `TIOCSWINSZ` ioctl). |
| Auth | Ephemeral `_SESSION_TOKEN` query param; server bound 127.0.0.1 by default. |
| Restart/resume | None — closing the WS terminates the child PTY (`hermes --tui`). |
| Error cases | Bad/expired token → connection refused. |
| Hazard | Encoding/Unicode width: PTY is byte-stream; no normalization. *(portability hazard — IME/emoji width)* |

### Gateway → Platform: `BasePlatformAdapter.send` / `edit_message`
*(observed fact — `gateway/platforms/telegram.py` + `gateway/stream_consumer.py`)*

| Field | Value |
|---|---|
| Producer | `gateway/run.py`, `GatewaySession`, `GatewayStreamConsumer` |
| Consumer | Concrete subclass of `BasePlatformAdapter` (Telegram, Discord, Slack, etc.) |
| Transport | Platform-specific (HTTPS, WebSocket, IMAP/SMTP, etc.). |
| Methods | `async send(chat_id, content, reply_to=None, metadata=None) -> SendResult`; `async edit_message(chat_id, message_id, content, *, finalize=False) -> SendResult`; `truncate_message(text, limit, len_fn=...) -> List[str]`; `format_message(text) -> str`; constant `MAX_MESSAGE_LENGTH` (Telegram 4096); optional `REQUIRES_EDIT_FINALIZE`; optional `delete_message(chat_id, message_id)`. |
| Required fields | `chat_id`, `content`. |
| Optional fields | `reply_to`, `metadata` (dict, platform-specific), `finalize`. |
| SendResult | `success: bool`, `message_id: Optional[str]`, `error: Optional[str]` *(strong inference — usage in stream_consumer)*. |
| Error cases | Whitespace-only `content` → `SendResult(success=True, message_id=None)`. Flood control matched by substrings `flood`/`retry after`/`rate` in `error`; backoff doubles `_current_edit_interval` (cap 10s); after 3 strikes edits permanently disabled and stream switches to fallback final-send. |
| Hazards | MarkdownV2 chunk indicator `(1/2)` must escape parens for Telegram (else raw fallback). `utf16_len` used for Telegram length. *(portability hazard — encoding)* |

### Gateway Stream Consumer events (sync→async bridge)
*(observed fact — `gateway/stream_consumer.py`)*

| Field | Value |
|---|---|
| Producer | `AIAgent` worker thread via `stream_delta_callback(text)` |
| Consumer | Async task `GatewayStreamConsumer.run()` |
| Transport | Thread-safe `queue.Queue` |
| Sentinels | `_DONE` (stream complete), `_NEW_SEGMENT` (tool-boundary, finalize+open new bubble), `_COMMENTARY` (interim assistant message). |
| Required fields | text delta string; `None` text means tool boundary. |
| Identifiers | `_message_id` (or `__no_edit__` sentinel for platforms that do not return ids); `_message_created_ts`. |
| Ordering | FIFO queue; consumer drains opportunistically; edit cadence governed by `edit_interval` (default 1.0s) + `buffer_threshold` (40 chars). |
| Error cases | Flood-control adaptive backoff (×2, cap 10s, 3 strikes). After exhaustion: fallback final-send; cursor stripped best-effort; `_flush_segment_tail_on_edit_failure` rescues unsent tail before reset. |
| Restart/resume | Per-turn instance — discarded after `finish()` and task await. |
| Hazard | `<think>`/`<reasoning>`/`<REASONING_SCRATCHPAD>` block filter has block-boundary state machine; partial-tag tail held back. *(portability hazard — tag list must stay in sync with `cli.py` `_OPEN_TAGS`/`_CLOSE_TAGS` and `run_agent.py` `_strip_think_blocks`)* |

### Gateway → AIAgent: `run_conversation()` invocation
*(observed fact — runtime-lifecycle.md)*

| Field | Value |
|---|---|
| Producer | `GatewaySession._handle_message_with_agent` |
| Consumer | `AIAgent.run_conversation(messages, **callbacks)` (synchronous, blocking) |
| Transport | In-process Python call; gateway runs it in a thread pool to avoid blocking the asyncio event loop. *(strong inference — sync loop wrapped in async gateway)* |
| Ordering | Strict per-session; `_pending_messages` queue holds inbound while a turn is in flight. |
| Identifiers | `task_id` per tool call; `session_id`, `session_key`. |
| Error cases | Iteration budget (default 90 + one-turn grace); interrupt via `_interrupt_requested`; provider exceptions classified by `agent/error_classifier.py`. |
| Restart/resume | `resume_pending` flag preserves `session_id` on next access; `suspend_session` forces fresh `session_id` on `/stop`. |

### Provider event stream (Chat Completions normalized)
*(observed fact — runtime-lifecycle.md + agent/anthropic_adapter.py mention)*

| Field | Value |
|---|---|
| Producer | `client.chat.completions.create(...)` per provider |
| Consumer | `run_conversation()` loop |
| Transport | HTTPS streamed (SSE) — provider-specific |
| Required fields | `messages[]` (`role`, `content`, possibly `tool_calls`, `tool_call_id`, `name`, `reasoning`); `tools[]` schemas; `model`. |
| Optional fields | `service_tier`, `reasoning_config`, `prefill_messages`, vendor extensions (Anthropic system blocks, Bedrock invocation params). |
| Identifiers | `tool_call.id` round-trips into the `role: "tool"` reply (`tool_call_id`). |
| Ordering | Sequential tool dispatch (NOT parallel) inside `run_conversation` even when provider returns parallel tool_calls. *(portability hazard — sequential vs. concurrent; reduces vendor-side parallelism)* |
| Error cases | Tenacity retries on transient provider errors; `nous_rate_guard` & `rate_limit_tracker` for quota; `credential_pool` rotates. |
| Restart/resume | Conversation re-loaded from `messages` table on next call; prompt-cache invariant requires NO mutation of past messages mid-conversation. |

### Tool dispatch — `model_tools.handle_function_call`
*(strong inference — runtime-lifecycle.md, AGENTS.md)*

| Field | Value |
|---|---|
| Producer | `run_conversation()` for each `tool_calls[i]` |
| Consumer | Registered handler in `tools.registry` |
| Transport | In-process Python call; result is a JSON string. |
| Required fields | `name`, `args` (JSON), `task_id`. |
| Optional | Pre/post tool-call hooks may inject side effects. |
| Identifiers | `task_id`, plus tool-internal IDs for delegate subagents and MCP calls. |
| Error cases | Handler exception → JSON error string returned (loop continues); guardrails in `agent/tool_guardrails.py`. |
| Hazard | Handler import order is significant: importing `model_tools.py` triggers plugin discovery as a side effect. *(portability hazard, AGENTS.md)* |

### Cron event — `cron/jobs.py` → `gateway/delivery.py`
*(observed fact — runtime-lifecycle.md, AGENTS.md, gateway/session.py `build_session_context_prompt`)*

| Field | Value |
|---|---|
| Producer | `cron/scheduler.py` (`croniter`) firing on schedule |
| Consumer | `gateway.delivery` → matching adapter `send` |
| Transport | In-process Queue → adapter HTTPS/etc. |
| Required fields | Job spec (`schedule`, `target`, `prompt`); `deliver` ∈ `origin` / `local` / `<platform>` / `<platform>:<chat_id>`. |
| Identifiers | Cron job id; resolved `HomeChannel(platform, chat_id, name)` if `deliver=<platform>`. |
| Ordering | Per-schedule firing; if a job is still running when next tick arrives behavior is unread. *(open question — overlap policy)* |
| Error cases | Delivery retried by adapter's flood/edit logic; `always_log_local=True` always also writes to `cron/output/`. |

### Trajectory append — `agent/trajectory.py` → `logs/session_*.json`
*(observed fact — runtime-lifecycle.md + AGENTS.md "Session search")*

| Field | Value |
|---|---|
| Producer | `AIAgent` post-turn writer; `batch_runner.py`; `trajectory_compressor.py`. |
| Consumer | `batch_runner`, `rl_cli.py`, `agent/curator.py`, RL `environments/`. |
| Transport | JSON file (whole-conversation trajectory) — full message list serialized with reasoning. |
| Required fields | `messages[]`, model, timestamps, token totals. |
| Ordering | Append-only per session (rewritten only by `/retry`, `/undo`, `/compress`). |
| Error cases | Corrupt JSON → file unreadable; SessionStore fallback prefers JSONL whichever has more messages. *(portability hazard — silent truncation guard in `SessionStore.load_transcript`)* |

### SessionStore — `gateway/sessions.json`
*(observed fact — `gateway/session.py` `SessionStore`)*

| Field | Value |
|---|---|
| Producer | `SessionStore._save()` (atomic-replace via `utils.atomic_replace`, fsync) |
| Consumer | `SessionStore._ensure_loaded`, `_load_sessions_index` (mcp_serve), `EventBridge._poll_once` |
| Transport | JSON file under `<HERMES_HOME>/sessions/sessions.json` |
| Required fields | per-key `SessionEntry.to_dict()` (session_key, session_id, created_at, updated_at, origin, …) |
| Ordering | mtime-based polling (200ms cycle in MCP `EventBridge`) |
| Error cases | Unknown platform values silently skipped on load. Atomic replace cleanup of temp file on failure. |
| Restart/resume | `resume_pending` / `suspended` / `expiry_finalized` survive restart. |

### Session-key schema (deterministic identifier)
*(observed fact — `build_session_key` in `gateway/session.py`)*

| Pattern | Trigger |
|---|---|
| `agent:main:<platform>:dm:<chat_id>` | DM with chat_id |
| `agent:main:<platform>:dm:<chat_id>:<thread_id>` | DM with thread |
| `agent:main:<platform>:dm:<thread_id>` | DM with no chat_id, thread fallback |
| `agent:main:<platform>:dm` | DM with no identifiers (single shared) |
| `agent:main:<platform>:<chat_type>:<chat_id>[:<thread_id>][:<participant>]` | Group/channel; participant appended only when isolation enabled and not in shared thread |
| WhatsApp normalization | `canonical_whatsapp_identifier()` applied to chat_id and participant_id to defeat JID/LID alias flips. *(observed fact)* |

### Approval request lifecycle
*(strong inference — `mcp_serve.permissions_*` + AGENTS.md slash commands)*

| Field | Value |
|---|---|
| Producer | `tools/tool_guardrails.py` / agent guard surfaces (e.g. shell-exec) |
| Consumer | CLI `/approve`/`/deny`; gateway `_pending_messages` interceptor; MCP `permissions_list_open` / `permissions_respond`. |
| Transport | In-process queue + (for MCP) cursor-stamped event. |
| States | `requested` → `approved (allow-once / allow-always)` / `denied`. Allow-always grants stick to subsequent calls. |
| Error cases | Unknown id → `{"error": "Approval not found"}`; invalid decision → JSON error. |
| Restart/resume | MCP-observed approvals are session-only. CLI persists approvals to disk *(open question — exact path)*. |

### Plugin lifecycle hooks
*(observed fact — AGENTS.md, architecture-map.md `plugins/`)*

| Field | Value |
|---|---|
| Producer | `model_tools.py` (pre/post tool), `run_agent.py` (pre/post llm), session boundary in `cli.py` / gateway. |
| Consumer | Plugin's `register(ctx)` returned hook callables. |
| Hooks | `pre_tool_call`, `post_tool_call`, `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`. |
| Transport | In-process callable invocation. |
| Ordering | Pre hooks run in registration order; post hooks in reverse. *(strong inference)* |
| Error cases | Hook exception logged; tool/LLM call still proceeds. *(strong inference — pluggable layer must be tolerant)* |

### OpenAI-compatible API server
*(strong inference — file size 125KB; arch-OQ5 left open for now)*

| Field | Value |
|---|---|
| Producer | OpenAI-API client (third-party). |
| Consumer | `gateway/platforms/api_server.py`. |
| Transport | HTTPS (FastAPI). |
| Auth | `API_SERVER_KEY` header + `API_SERVER_HOST` allow-list. |
| Surface | `/v1/chat/completions` (streaming + non-streaming), `/v1/models`. *(strong inference — name "OpenAI-compatible")* |
| Restart/resume | Each request is a new conversation OR routed to a known session_id depending on body — *(open question)*. |

### Webhook receiver
*(observed fact — `gateway/platforms/webhook.py` listed)*

| Field | Value |
|---|---|
| Producer | External HTTP POSTer. |
| Consumer | `gateway/platforms/webhook.py`. |
| Transport | HTTP. |
| Auth | Configured per-channel. |
| Restart/resume | Each request is independent. |

### Telegram webhook (alt to long-poll)
*(observed fact — `.env.example` `TELEGRAM_WEBHOOK_*`)*

| Field | Value |
|---|---|
| Transport | HTTPS POST from Telegram to `TELEGRAM_WEBHOOK_URL` on `TELEGRAM_WEBHOOK_PORT` |
| Auth | `TELEGRAM_WEBHOOK_SECRET` header validation |
| Ordering | Telegram delivers in arrival order; duplicate-update_id dedup is the receiver's responsibility. *(strong inference)* |

### Slack Socket Mode events
*(observed fact — pyproject `slack-bolt` + `[slack]` extra)*

| Field | Value |
|---|---|
| Transport | WebSocket via `slack-bolt` (Socket Mode) |
| Slash subcommands | `/hermes <subcommand>` advertised in README |
| Auth | `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` |

### Slash command catalog (in-conversation control plane)
*(observed fact — README, AGENTS.md, gateway `reset_triggers`)*

| Field | Value |
|---|---|
| Producer | User typing `/<name>` in any chat surface |
| Consumer | `hermes_cli/commands.COMMAND_REGISTRY` central dispatcher; gateway command interceptor for `/stop`, `/new`, `/queue`, `/status`, `/approve`, `/deny`, `/platforms`, `/sethome`. |
| Transport | In-process. |
| Reset triggers | Configurable; default `["/new", "/reset"]`. |
| Hazard | Two-guard interaction (`_pending_messages` + command interceptor): every new control command MUST bypass both guards. *(portability hazard — AGENTS.md)* |

## State Machines

### 1. Gateway listener lifecycle (per-platform adapter) [resolves arch-CF4]

| Current State | Event / Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| `unconfigured` | adapter constructor | env vars + `gateway.config` enable platform | `configuring` | reads tokens, validates settings |
| `configuring` | `acquire_scoped_lock(token)` from `gateway.status` | profile-scoped, no other process holds it | `connecting` | scoped lock held |
| `configuring` | lock contention | another profile holds the same bot token | `unconfigured` (rejected) | error logged; adapter not started |
| `connecting` | `adapter.connect()` / `start()` returns | network reachable | `connected` | platform-side login + listener handle |
| `connected` | inbound message arrives | session_key not in `_active_sessions` | `dispatching` | enqueue or directly dispatch |
| `connected` | inbound message arrives | session_key in `_active_sessions` | `connected` (queued) | append to `_pending_messages` |
| `connected` | command interceptor matches `/stop`/`/new`/`/queue`/`/status`/`/approve`/`/deny`/`/platforms`/`/sethome` | command word matches before agent dispatch | `connected` | execute control op; bypass agent |
| `connected` | flood-control / transport error during edit | `_flood_strikes < 3` | `connected (degraded)` | `_current_edit_interval *= 2`, cap 10s |
| `connected (degraded)` | 3rd flood strike | `_edit_supported = False` | `connected (fallback)` | `_fallback_final_send = True`; cursor strip best-effort |
| `connected` | shutdown signal | drain timeout exceeded | `draining (drain_timeout)` | mark `resume_pending=True` for active sessions |
| `connected` | shutdown signal | clean | `disconnecting` | release scoped lock, `disconnect()` |
| `disconnecting` | adapter cleanup done | — | `disconnected` | platform connection closed |

### 2. Per-conversation `SessionEntry` lifecycle (resolves arch-CF4 + CF3)

| Current State | Event | Guard | Next State | Side Effects |
|---|---|---|---|---|
| (none) | `get_or_create_session(source)` | session_key not in `_entries` | `active` | create `SessionEntry`, mint `session_id = YYYYMMDD_HHMMSS_<uuid8>`, save sessions.json, `SessionDB.create_session` |
| `active` | next inbound message | `_should_reset` returns None | `active` | `updated_at = now`, save |
| `active` | reset policy fires | mode in (`idle`,`both`) and `now > updated_at + idle_minutes` | `active (auto-reset)` | new `session_id`; old ended in DB with `end_reason="session_reset"`; `was_auto_reset=True`, `auto_reset_reason ∈ {"idle","daily"}` |
| `active` | reset policy fires | mode in (`daily`,`both`) and `updated_at < today_reset(at_hour)` | `active (auto-reset)` | as above with reason `"daily"` |
| `active` | `/stop` (suspend_session) | — | `suspended` | persist `suspended=True`; next access auto-resets |
| `active` | gateway drain timeout | not `suspended` | `resume_pending` | persist `resume_pending=True`, `resume_reason`, `last_resume_marked_at`; `session_id` preserved |
| `resume_pending` | next inbound message | not `suspended` | `active` (same session_id) | `updated_at=now`; `clear_resume_pending` after successful turn |
| `resume_pending` | startup `suspend_recently_active(120)` | `updated_at >= cutoff` and not `resume_pending` | (skipped — only non-pending sessions are bulk-suspended) | — |
| `active` | `/reset` (`reset_session`) | session_key exists | `active (fresh)` | new session_id, `is_fresh_reset=True` |
| `active` | `/resume <id>` (`switch_session`) | target session_id exists in DB | `active (target_id)` | end-session current; reopen target |
| `active` | background-process tied to session | `has_active_processes_fn(key)=True` | `active` (immune) | reset/prune skipped while process is active |
| `active` | `prune_old_entries` | `updated_at < now - max_age_days` and not suspended and no active process | (deleted) | `_entries` removed; transcript stays in SQLite |

Reset policy modes: `none`, `idle`, `daily`, `both`. `at_hour` 0–23 (default 4); `idle_minutes` default 1440. `notify_exclude_platforms = ("api_server","webhook")` skip the user-facing notice. *(observed fact — `SessionResetPolicy`)*

### 3. Agent turn lifecycle (`run_conversation`) [resolves part of arch-OQ1]

| Current State | Event | Guard | Next State | Side Effects |
|---|---|---|---|---|
| `idle` | message arrives | budget remaining | `pre-llm` | run `pre_llm_call` plugin hooks |
| `pre-llm` | hook done | not interrupted | `awaiting_provider` | provider call begins (blocking) |
| `awaiting_provider` | response with no tool_calls | — | `final` | run `post_llm_call`; return content |
| `awaiting_provider` | response with tool_calls | — | `dispatching_tool` | i=0 |
| `dispatching_tool` | tool i complete | i+1 < len(tool_calls) | `dispatching_tool` (i+1) | append tool_result message |
| `dispatching_tool` | last tool complete | `api_call_count < max_iterations` and budget remains | `pre-llm` | next iteration |
| `dispatching_tool` | tool requires approval | guardrail | `awaiting_approval` | emit approval request |
| `awaiting_approval` | `/approve`/`/deny` | — | `dispatching_tool` or `final (denied)` | resume or abort |
| any | `_interrupt_requested = True` | — | `interrupted` | break loop, return partial |
| `pre-llm`/`dispatching_tool` | `iteration_budget.remaining == 0` | one-turn grace flag | `final (budget)` | exit loop |
| `awaiting_provider` | provider exception | classifier=transient | `awaiting_provider` (retry via tenacity) | rotate via `credential_pool` if quota |
| `awaiting_provider` | provider exception | classifier=fatal | `error` | `error_callback` fires |

Synchronous barriers: provider call (blocking), tool call (sequential, not parallel). Observational events: `stream_delta_callback` deltas, `tool_callback` notifications, plugin hooks. *(observed fact — runtime-lifecycle.md)*

### 4. Stream consumer lifecycle (per turn)

| Current State | Event | Guard | Next State | Side Effects |
|---|---|---|---|---|
| `init` | first delta | `_message_id is None` and visible chars >= 4 (gate) | `editing` | adapter.send → store `_message_id`, `_message_created_ts`; notify `_on_new_message` |
| `editing` | delta or interval tick | `len(accumulated) <= safe_limit` | `editing` | `adapter.edit_message` |
| `editing` | overflow | `len(accumulated) > safe_limit` | `splitting` | edit first chunk; new send for remainder; reset `_message_id` for next |
| `editing` | `_NEW_SEGMENT` (tool boundary) | preserve `__no_edit__` if applicable | `editing` (fresh msg) | finalize current; rescue tail if edit failed |
| `editing` | `_COMMENTARY` | text non-empty | `editing` (commentary sent) | adapter.send; do NOT set `_already_sent` |
| `editing` | flood error | `_flood_strikes < 3` | `editing (degraded)` | interval *= 2, cap 10s |
| `editing (degraded)` | 3rd flood strike | — | `fallback` | `_edit_supported=False`; strip cursor best-effort |
| `editing` | `_DONE` and adapter requires finalize | `_adapter_requires_finalize` | `done (finalize)` | `edit_message(finalize=True)` |
| `editing` | `_DONE` long-lived preview | `fresh_final_after_seconds > 0 and age >= threshold` | `done (fresh-final)` | new send; best-effort delete preview |
| `fallback` | `_DONE` | continuation non-empty | `done (fallback)` | `_send_fallback_final` chunked send with one retry on flood |
| any | `asyncio.CancelledError` | accumulated text + message_id | `done (best-effort)` | one final edit; promote `final_response_sent` only if it succeeded |

Think-block filter is a sub-state machine: `outside_think | inside_think | partial_tag_held` driven by tag scan. *(observed fact)*

### 5. Tool call lifecycle (per `tool_call`)

| Current State | Event | Guard | Next State | Side Effects |
|---|---|---|---|---|
| `received` | `handle_function_call(name, args, task_id)` | tool registered | `pre-hook` | run `pre_tool_call(name, args, ctx)` |
| `pre-hook` | hook done | no abort | `executing` | handler invoked |
| `executing` | handler returns string | — | `post-hook` | run `post_tool_call(name, result)` |
| `post-hook` | done | — | `complete` | append `tool_result` message |
| `executing` | handler raises | — | `error` | JSON `{"error": str(exc)}` returned to model |
| `pre-hook` | guardrail denies | tool requires approval | `awaiting_approval` | enqueue approval request |

### 6. MCP EventBridge poll loop

| Current State | Event | Guard | Next State | Side Effects |
|---|---|---|---|---|
| `polling` | tick (200ms) | `sessions.json mtime` changed | `polling` | reload `_cached_sessions_index` |
| `polling` | tick | both mtimes unchanged | `polling` | no-op (free) |
| `polling` | new message in DB | `ts > last_seen` and role in {user, assistant} | `polling` | enqueue `QueueEvent(type="message")`, set `_new_event` |
| `polling` | queue size > 1000 | — | `polling` | FIFO trim |
| `wait_for_event` | matching event found | — | returns | — |
| `wait_for_event` | timeout | — | returns `{"event": null, "reason":"timeout"}` | — |

## Persistent Schema Notes [resolves arch-CF3]

### A. SQLite session DB — `hermes_state.SessionDB` (`<HERMES_HOME>/state.db`)

*(observed fact — `hermes_state.SCHEMA_SQL`, schema_version=11)*

- **Mode:** WAL (concurrent readers + one writer). One DB file shared by CLI, gateway, and any subprocess on the same `HERMES_HOME`.
- **Tables:**
  - `schema_version(version INTEGER)` — single row, current value 11. Migrations stepwise via internal `_migrate_*` functions. *(strong inference — code style)*
  - `sessions(id PK, source, user_id, model, model_config, system_prompt, parent_session_id FK→sessions, started_at REAL, ended_at REAL, end_reason, message_count, tool_call_count, input_tokens, output_tokens, cache_read_tokens, cache_write_tokens, reasoning_tokens, billing_provider, billing_base_url, billing_mode, estimated_cost_usd, actual_cost_usd, cost_status, cost_source, pricing_version, title, api_call_count)`.
  - `messages(id PK AUTOINCREMENT, session_id FK→sessions, role, content, tool_call_id, tool_calls, tool_name, timestamp REAL, token_count, finish_reason, reasoning, reasoning_content, reasoning_details, codex_reasoning_items, codex_message_items)`.
  - `state_meta(key PK, value)` — KV.
  - FTS5 virtual table over `messages.content` (referenced by AGENTS.md). *(observed fact — file docstring)*
- **Indices:** `idx_sessions_source(source)`, `idx_sessions_parent(parent_session_id)`, `idx_sessions_started(started_at DESC)`, `idx_messages_session(session_id, timestamp)`.
- **Append-only?** `messages` is append-only EXCEPT for `replace_messages(session_id, messages)` (called by `/retry`, `/undo`, `/compress`). `sessions` is mutable for token totals + lifecycle fields. `parent_session_id` chains preserve compression-split history. *(observed fact)*
- **Branching:** linear-by-default; compression creates a new session_id with `parent_session_id` pointing to the prior session (compression-triggered split).
- **Replay/resume:** `get_messages_as_conversation(session_id)` reconstructs full message list for next turn. `reopen_session(session_id)` clears `ended_at` to support `/resume`.
- **Locking/dedup:** SQLite WAL gives single-writer; `threading.Lock` in `SessionDB`. Read path tolerates corruption by skipping unparseable rows. *(strong inference)*
- **Hazard:** `load_transcript` prefers JSONL when `len(jsonl) > len(db_messages)` to avoid silently truncating legacy sessions whose first post-migration turn appended only deltas. *(observed fact — `gateway/session.py`)*

### B. Sessions index — `<HERMES_HOME>/sessions/sessions.json`

*(observed fact — `SessionStore._save`)*

- JSON object: `session_key → SessionEntry.to_dict()`.
- Atomic writes: `tempfile.mkstemp` + `f.flush()` + `os.fsync()` + `utils.atomic_replace(tmp, target)`. Crash-safe.
- Mutable: rewritten on every state change (high write rate; mtime is the polling driver in MCP `EventBridge`).
- Survives restart fields: `suspended`, `resume_pending`, `resume_reason`, `last_resume_marked_at`, `expiry_finalized`, `is_fresh_reset`, token totals, cost totals.

### C. Legacy JSONL transcripts — `<HERMES_HOME>/sessions/<session_id>.jsonl`

*(observed fact — `SessionStore.append_to_transcript`, `rewrite_transcript`)*

- One JSON message per line, UTF-8, `ensure_ascii=False`.
- Append-only EXCEPT `/retry`, `/undo`, `/compress` rewrite via `rewrite_transcript`.
- Written redundantly with SQLite (`skip_db=True` flag prevents the duplicate-write `_flush_messages_to_session_db` bug #860).
- Read fallback: corrupt lines skipped with WARNING.
- Migration path: legacy sessions retain JSONL as canonical until first post-migration write reaches SQLite parity.

### D. Trajectory JSON — `<HERMES_HOME>/logs/session_YYYYMMDD_HHMMSS_UUID.json`

*(observed fact — runtime-lifecycle.md, AGENTS.md)*

- Whole-conversation snapshot with `messages[]`, model, timings, token totals, reasoning where applicable.
- Generated when `save_trajectories=True`. Consumed by `batch_runner`, `rl_cli`, `trajectory_compressor`, `agent/curator`.
- Append-only; rewritten only on manual edit. Used as RL training corpus.

### E. Prompt-cache layout

*(observed fact — AGENTS.md, runtime-lifecycle.md)*

- System prompt is **cache-stable**: `agent/prompt_builder.py` arranges it so identical prefixes hit Anthropic / OpenRouter prompt cache.
- Mid-conversation invariant: do NOT alter past context, change toolsets, or rebuild system prompts mid-conversation. Only legitimate mutation point: `agent/context_compressor.py` triggered at `compression.threshold` (default 0.85).
- Storage: implicit (provider-side cache). Hermes simply maintains the invariant.
- *(portability hazard — provider-specific cache key derivation; cache cost tracking via `cache_read_tokens` + `cache_write_tokens` fields in `sessions`)*

### F. Kanban DB — `kanban_db.py` (SQLite)

- Schema not directly read this phase. Carry forward to defect-scan-semantic if a defect is suspected. *(open question — schema version, table names)*

### G. asyncpg / Postgres mode

- pyproject extra `[gateway]` includes `asyncpg`. *(observed fact)*
- Used as an alternative to SQLite for the gateway sessions index when configured. Schema mirrors `sessions.json`/`SessionDB`. *(strong inference; not directly read)*
- *(open question — exact migration path / schema parity)*

### H. MCP OAuth tokens — `tools/mcp_oauth_manager.py`

*(observed fact — state-and-storage.md)*

- Per-MCP-server credentials stored as JSON under HERMES_HOME.
- Refresh flow uses standard OAuth2 refresh-token semantics.
- *(portability hazard — refresh-on-401 reentrancy; clock skew on token expiry)*

### I. Channel directory — `<HERMES_HOME>/channel_directory.json`

*(observed fact — `mcp_serve._load_channel_directory`)*

- Cached list of `platform → [{id|chat_id, name|display_name, type}]` for `channels_list` MCP tool.
- Mutable; refreshed by gateway as platform memberships change. *(strong inference)*

### J. Sticker cache, browser session state, Honcho/Qwen creds, Playwright browsers

- Filesystem caches; no schema. Documented in state-and-storage.md.

## Compatibility Hazards

| Hazard | Where It Appears | Severity | Notes |
|---|---|---|---|
| ANSI terminal semantics | `cli.py` `prompt_toolkit.patch_stdout` reportedly leaks `\033[K` as literal text (AGENTS.md known pitfall, arch-CF7); `KawaiiSpinner` uses ANSI; PTY bytestream over `/api/pty` is unfiltered | High | Any port must replicate the patch_stdout fix or drop the spinner; raw ANSI must be sanitized before logs/transcripts. |
| Encoding / Unicode width | Telegram message length uses `utf16_len`; cursor character `▉` reported to render as "tofu" on some clients (`_MIN_NEW_MSG_CHARS = 4` guard); MarkdownV2 chunk markers `(1/2)` need backslash-escape | High | Per-platform length helpers must be preserved; control-character handling not portable across runtimes. |
| Async-vs-sync ordering | Sync `run_conversation` invoked inside async gateway; tool dispatch is sequential despite providers returning parallel `tool_calls` | High | Reimplementations must NOT eagerly parallelize tool dispatch — the prompt-cache invariant and tool side-effect order would break. |
| Filesystem locking assumptions | `SessionDB` uses SQLite WAL with single-writer; `gateway/sessions.json` uses `tempfile.mkstemp` + `os.fsync` + `atomic_replace`; `gateway.status.acquire_scoped_lock` is profile-token-scoped (mechanism not directly read) | Medium | Windows is not supported; WSL2 only. Atomic rename semantics differ on FAT/exFAT/NTFS. |
| OAuth token refresh | MCP OAuth manager; per-platform tokens; Qwen reuses `~/.qwen/oauth_creds.json`; Honcho `~/.honcho/config.json`; auth.py 183KB | Medium | Refresh-on-401 reentrancy; clock skew on token expiry; concurrent refreshers can race. |
| Shell quoting / OS differences | Terminal backends (local/docker/ssh/modal/daytona/singularity) shell out; `pty_bridge.py` uses `ptyprocess` | High | Native Windows unsupported; Termux uses curated `[termux]` extra. |
| Process-global state mutation | `_last_resolved_tool_names` in `model_tools.py` saved/restored by `delegate_tool._run_single_child` | High | Re-entrancy bug surface; thread/asyncio task safety unclear. Not portable to a single-globals model. |
| Implicit import-side-effect ordering | Tool registry populated by `tools/*.py` import-time `register()` calls; plugin discovery side-effect of importing `model_tools.py` | High | A port must make discovery explicit. AGENTS.md documents this as a known pitfall. |
| Two-guard message dispatch | `_pending_messages` (`gateway/platforms/base.py`) + command interceptor (`gateway/run.py`) — every new control command MUST bypass BOTH | High | Concurrency contract violation surface. Routed to defect-scan-semantic (arch-CF8). |
| Prompt-cache mutation invariant | Slash commands or compaction code paths could alter past context, busting cache | Medium | Routed to defect-scan-semantic (arch-CF9). |
| Think-block filter tag list | `<think>`, `<reasoning>`, `<REASONING_SCRATCHPAD>`, `<THINKING>`, `<thinking>`, `<thought>` in three places: `cli.py _OPEN_TAGS`, `run_agent.py _strip_think_blocks`, `gateway/stream_consumer.py` | Medium | Three-way duplication of the tag list; ports must consolidate. |
| Stream consumer flood-control heuristic | Substring match `flood`/`retry after`/`rate` in `error` field | Medium | English-only; provider-specific error text changes will silently break. |
| WhatsApp identity normalization | `canonical_whatsapp_identifier()` defeats Baileys JID/LID alias flips that otherwise split a single user across two sessions | Medium | Critical for session-key stability; ports must replicate. |
| Time / TZ assumptions | Reset policy `at_hour` evaluated against `datetime.now()` (local time, NOT UTC) | Medium | `notify_exclude_platforms = ("api_server","webhook")` skip notice; tests pin TZ=UTC and may mask the local-time choice. |
| Provider stream framing | Edit-transport (send-then-edit) is "universally supported" across Telegram/Discord/Slack but each platform's `MAX_MESSAGE_LENGTH` and edit limits differ | Medium | A port must keep the per-adapter constants and length helpers. |
| MCP polling-based liveness | `EventBridge` polls SQLite + sessions.json every 200ms (mtime-cached). No push channel. | Low | A port could substitute push (FS watch / DB notify) but the cursor-monotonic contract must be preserved. |
| Session key collision under missing identifiers | `agent:main:<platform>:dm` is the bare-fallback key for DMs without chat_id or thread_id — multiple users could share | Medium | Documented in `build_session_key` docstring; safety relies on platform always providing chat_id. |
| ACP probe stderr filter | `_BenignProbeMethodFilter` suppresses tracebacks for unknown probe methods (`ping`, `health`, `healthcheck`) | Low | Editor-specific liveness checks vary; a new probe name would re-introduce noise. |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| prot-OQ1 | open question | Cron-job overlap policy when next tick fires while a job is still running. | Not visible in `cron/jobs.py` without reading it; deferred to keep within budget. |
| prot-OQ2 | open question | Exact OpenAI-compatible API server schema (`gateway/platforms/api_server.py`, 125KB) — does it accept session_id continuation, and how does it map to gateway sessions? Also continues arch-OQ5. | File too large for budget. |
| prot-OQ3 | open question | asyncpg/Postgres alternative gateway store — exact migrations and parity with SQLite path. | Code path not directly read. |
| prot-OQ4 | open question | Plugin hook ordering across multiple plugins (registration vs. priority); error containment policy when one hook raises. | Architecture-level evidence only; precise dispatch loop not read. |
| prot-OQ5 | open question | Kanban DB schema and migration table. | Not read this phase. |
| prot-OQ6 | open question | Approval persistence path on the CLI — does `/approve` persist allow-always grants across restart, and where? | MCP-side is session-only (observed); CLI-side not read. |
| prot-OQ7 | open question | Exact send-side dedup for Telegram webhook (update_id) and Slack Socket Mode reconnect-replay behavior. | Not directly read. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| prot-CF1 | defect-scan-semantic | Verify two-guard invariant: every control command path (`/stop`,`/new`,`/queue`,`/status`,`/approve`,`/deny`,`/platforms`,`/sethome`) bypasses both `_pending_messages` queue and `gateway/run.py` command interceptor. (extends arch-CF8) | Concurrency-contract verification. |
| prot-CF2 | defect-scan-semantic | Audit prompt-cache invariant: confirm no slash command or compaction path mutates past `messages` mid-conversation. Inspect `/compress`, `/undo`, `/retry`, plugin hooks. (extends arch-CF9) | Semantic invariant audit. |
| prot-CF3 | defect-scan-semantic | Race between `replace_messages` (SQLite) and `rewrite_transcript` (JSONL) in `gateway/session.py`: SQLite is "atomic", JSONL is open-truncate-write — partial JSONL on crash. | Concurrency / durability. |
| prot-CF4 | defect-scan-semantic | Sequential tool dispatch ordering vs. provider-side parallel `tool_calls`: ensure no tool depends on still-pending sibling outputs in the same response. | Semantic ordering audit. |
| prot-CF5 | defect-scan-semantic | Stream consumer cancellation path: `_best_effort_ok` final flag promotion logic and prevention of duplicate gateway final-send. | Concurrency invariant. |
| prot-CF6 | porting | Reimplementation must consolidate the think-block tag list (currently in 3 places) and provide a single filter primitive. | Porting decomposition. |
| prot-CF7 | porting | Per-adapter `MAX_MESSAGE_LENGTH`, `format_message`, `truncate_message`, `REQUIRES_EDIT_FINALIZE` constitute a stable adapter contract; codify it in the port. | Porting contract. |
| prot-CF8 | porting | Decide whether to keep mtime-polling (200ms) for the MCP event bridge or switch to push (FS watch / DB notify). | Porting design choice. |
| prot-CF9 | reimplementation-spec | Decide if `parent_session_id` chains (compression splits) remain linear or move to a branch model in the rewrite. | Strategic decision. |
| prot-CF10 | reimplementation-spec | Reset policy semantics: keep "at_hour in local time" or move to UTC + per-user TZ. | Strategic decision affecting user-visible behavior. |

---

(Validation block appended in a follow-up commit per VALIDATE.md.)
