## 2026-05-06 — porting phase

**Build/deploy tiering for the port:**

- **core:** single-binary or single-package install for the kernel + CLI + MCP + ACP, `~/.hermes/.env` token store with config precedence env > .env > config.yaml > DEFAULT_CONFIG (NEVER mutate process env from YAML — closes D6.1 CRITICAL), one config loader (closes D6.2). Rationale: install ergonomics and config correctness are first-port-blockers.
- **important:** per-platform adapter packaging (Telegram/Discord first), explicit subprocess env allowlist (closes D4.2 `*_PROXY` leak), reproducible startup with explicit plugin registration (closes D1.x/D6.8 import-side-effect). Rationale: required for safe deployment of gateway adapters and reliable startup performance.
- **optional:** dashboard SPA build pipeline, web `/api/pty` server, RL environments. Rationale: large incremental build surface; defer to second port.
- **incidental:** the 167 KB `auxiliary_client.py` monolith. Rationale: do not port as-is; decompose into 5 modules (provider-call dispatch, retry policy, credential rotation, token accounting, error classification) per arch-CF10 / port-CF4.
