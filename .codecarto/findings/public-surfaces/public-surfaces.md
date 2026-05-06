## 2026-05-06 — porting phase

**Surface tiering for the port (resolves arch-CF11 partial):**

- **core:** CLI (`hermes`), MCP server (stdio FastMCP, 10 tools, cursor-monotonic 200ms poll), ACP adapter (stdio JSON-RPC), Telegram, Discord. Rationale: smallest surface area that delivers a usable agent on dev machines + 2 chat platforms with simple bot-token auth. ACP/MCP are the smallest stable contracts and unlock IDE/agent-client integration.
- **important:** TUI (`hermes --tui`), Slack, WhatsApp, TUI gateway JSON-RPC (27+ methods). Rationale: TUI replaces CLI for power users; Slack/WhatsApp expand chat reach but introduce OAuth refresh races (D2.x) and webhook surface respectively.
- **optional:** Web dashboard SPA, `/api/pty` WebSocket, Matrix, Signal, Teams, Feishu, Yuanbao, Email, generic api_server, generic webhook. Rationale: large incremental surface for marginal user gain in a first port; ANSI-over-WebSocket PTY is a portability hazard worth deferring.
- **incidental:** RL environments / trajectory JSON, legacy JSONL transcript *write* path (read path is important for migration). Rationale: source-specific ergonomics; not required for parity on user-visible workflows.
