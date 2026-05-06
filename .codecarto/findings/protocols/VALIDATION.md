# Validation — protocols phase

This file is a **sidecar validation block** for `protocols-and-state.md`. The framework convention is to append the Validation block at the end of the primary output, but three subagent attempts at the 45 KB inline rewrite timed out with stream-idle errors. The orchestrator placed the validation content here as a documented compromise. See `BACKLOG.md` and `DECISIONS.md` (D8) for context.

The primary file `protocols-and-state.md` ends with the placeholder line `(Validation block appended in a follow-up commit per VALIDATE.md.)`. The validation table below replaces what would have been written there.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | An event catalog is documented. | PASS | `protocols-and-state.md` §Event Catalog enumerates 19+ events: MCP Server (10 tools, stdio FastMCP); ACP Adapter (stdio JSON-RPC); TUI Gateway (27+ JSON-RPC methods); Dashboard PTY WebSocket; Gateway→Platform send/edit_message; Stream Consumer events (sync→async bridge); Gateway→AIAgent run_conversation invocation; Provider event stream; Tool dispatch; Cron event; Trajectory append; SessionStore (sessions.json); Session-key schema; Approval lifecycle; Plugin lifecycle hooks; OpenAI-compatible API server; Webhook receiver; Telegram webhook; Slack Socket Mode; Slash command catalog. |
| 2 | A state machine is documented. | PASS | `protocols-and-state.md` §State Machines documents 6: (1) gateway listener lifecycle (resolves arch-CF4); (2) per-conversation SessionEntry lifecycle (resolves arch-CF4 + CF3); (3) agent turn lifecycle `run_conversation` (resolves part of arch-OQ1); (4) stream consumer lifecycle (per turn); (5) tool call lifecycle; (6) MCP EventBridge poll loop. |
| 3 | Persistent schema notes are documented. | PASS | `protocols-and-state.md` §Persistent Schema Notes (A–J) covers: SQLite SessionDB schema_version=11 with full DDL; sessions.json index (atomic-replace, fsync); legacy JSONL transcripts (length-preferred read); trajectory JSON; prompt-cache layout (provider-side, cache-stable system prompt); Kanban DB (deferred); asyncpg/Postgres mode; MCP OAuth tokens; channel directory; sticker/browser/auth caches. Resolves arch-CF3. |
| 4 | Compatibility hazards are documented. | PASS | `protocols-and-state.md` §Compatibility Hazards is an 18-row table covering: ANSI terminal semantics; Encoding/Unicode width; Async-vs-sync ordering; Filesystem locking; OAuth token refresh; Shell quoting/OS differences; Process-global state mutation; Implicit import-side-effect ordering; Two-guard message dispatch; Prompt-cache mutation invariant; Think-block filter tag list; Stream consumer flood-control heuristic; WhatsApp identity normalization; Time/TZ assumptions; Provider stream framing; MCP polling-based liveness; Session key collision under missing identifiers; ACP probe stderr filter. |
| 5 | Findings are marked with evidence levels. | PASS | Every claim throughout `protocols-and-state.md` is annotated with one of `observed fact`, `strong inference`, `portability hazard`, or `open question`. Evidence-level legend stated at top of the document. |

**Validated by:** 2026-05-06 — protocols phase implementing session (validation block applied by orchestrator as sidecar after three subagent timeouts on the 45 KB inline rewrite)
**Overall:** PASS

---

## Compliance note

The framework's strict reading of VALIDATE.md is that the validation block belongs **inline at the end of the primary output**. This sidecar is a documented deviation, tracked in:

- `BACKLOG.md` — known compliance gap, deferred to a future session that can do the 45 KB rewrite
- `DECISIONS.md` D8 — rationale for the deferral
- `CONVENTIONS.md` C3 — pattern for orchestrator-recoverable validation block writes

Closing this gap requires one `mcp__github__create_or_update_file` call rewriting `protocols-and-state.md` with the trailing placeholder replaced by the table above.
