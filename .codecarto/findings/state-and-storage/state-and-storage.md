## 2026-05-06 — porting phase

**Storage tiering for the port:**

- **core:** SessionDB logical schema (sessions, messages, state_meta, FTS5-equivalent message search) at schema_version=11; sessions.json index with atomic-replace write semantics; prompt-cache stable system-prompt prefix. Rationale: these define what "a session" means; users expect resume/save/search to work identically. The port may swap SQLite for another backend but must preserve the *logical* schema (port-CF6).
- **important:** legacy JSONL transcripts read path (length-preferred when present), per-session approval/deny state, message-edit history. Rationale: needed for migration of existing user data and parity on /undo, /retry, /save, /resume.
- **optional:** trajectory JSON export (RL training format), JSONL transcript *write* path. Rationale: optional for first port; can be added later without schema migration.
- **incidental:** Windows-specific FS-locking workarounds (currently WSL2-only). Rationale: a port should target POSIX-atomic-replace + Windows-atomic-rename-with-retry from day one; the source's WSL2-only stance is an implementation accident.
