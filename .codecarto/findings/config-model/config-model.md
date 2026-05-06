## 2026-05-06 … porting phase

**Config model tiering for the port:**

- **core:** single config loader implementing precedence env > .env file > config.yaml > DEFAULT_CONFIG; per-platform tokens via `~/.hermes/.env`; auth flips `GATEWAY_ALLOW_ALL_USERS` / `TEAMS_ALLOW_ALL_USERS` (one env var flips default-deny → default-allow — surfaced as a security-sensitive footgun, fix-before-porting D4.x). Rationale: config correctness is a CRITICAL invariant (D6.1 + D6.2); the port must not reproduce three loaders or yaml→os.environ mutation.
- **important:** model selection (`/model` slash command), tool approval allowlist with override flags (D4.x: re-derive default-deny in port), per-adapter `MAX_MESSAGE_LENGTH` / `format_message` / `truncate_message` / `REQUIRES_EDIT_FINALIZE` defaults (ctr-CF9+prot-CF7 codified as a typed interface). Rationale: required for parity on tool-execution and gateway behavior.
- **optional:** dashboard SPA settings, RL training hyperparameters, optional adapter token sets (Matrix/Signal/Teams/Feishu/Yuanbao/Email/api_server/webhook). Rationale: large config surface for limited first-port value.
- **incidental:** YAML→os.environ mutation pattern (D6.1 CRITICAL), the existence of three separate config-loader code paths (D6.2). Rationale: implementation accidents; the port replaces both with one loader and zero env mutation.
