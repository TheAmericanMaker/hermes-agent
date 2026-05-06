# Reverse-Engineering Bundle

*Project: hermes-agent (Nous Research). Phase: porting (Phase 6 of pipeline-full-with-deep-audit). Date: 2026-05-06.*

Evidence-level legend (used inline): `[OF]` observed fact, `[SI]` strong inference, `[PH]` portability hazard, `[OQ]` open question.

## System Summary

hermes-agent is a self-improving AI agent runtime written in Python 3.11+. It is a single repository (~200K LOC, 11 top-level packages) that composes a provider-agnostic agent kernel with a large catalogue of tools, skills, scheduled cron tasks, and reinforcement-learning environments, and it surfaces the resulting agent through six different *delivery surfaces* simultaneously: a CLI, an interactive TUI, a web dashboard SPA + API, a multi-platform messaging gateway with 18+ chat-platform adapters, and two machine protocols (MCP for Claude/IDE clients, ACP for Agent Client Protocol consumers). `[OF]`

The agent kernel is a turn-loop that compiles a system prompt, calls a normalized provider adapter, parses tool calls, executes them through a tool registry (with hooks, transforms, and an approval workflow), compresses long contexts via a curator, and persists messages to a SQLite SessionDB plus legacy JSONL transcripts and a JSON sessions index. Around this kernel the gateway dispatches platform messages, the cron layer schedules background tasks, the dashboard streams ANSI-rendered terminal output over a WebSocket PTY, and the MCP server exposes tools to external clients. `[SI]`

The system is a *layered composition*: a portable core (agent kernel + tool registry + provider normalization + storage) wrapped by adapter shells. A clean port can ship a viable subset (CLI + MCP + ACP + Telegram + Discord) without re-implementing every shell. The hardest portability hazards are not algorithms, they are *invariants* the source enforces implicitly: cursor-monotonic MCP polling, atomic-replace sessions.json, prompt-cache stability, two-guard message dispatch, and several global-mutation patterns that any port must consciously redesign. `[SI][PH]`

## Layer Map With Ownership

Concept-named layers (top = lowest reusable, bottom = surface). `[SI]`

| Layer / Module | Role | Owns (source packages) |
|---|---|---|
| Agent Kernel | Turn-loop, prompt builder, context compressor, curator, model adapter dispatch | `agent/` |
| Provider Normalization | One call shape across vendors; retry, credential rotation, token accounting, error classification | `agent/auxiliary_client.py` (167KB monolith — flagged for decomposition, see arch-CF10) |
| Tool Registry & Execution | 70+ tools, hooks, transforms, approval/deny gating | `tools/`, `skills/` |
| Storage & Session | SessionDB (SQLite v11 schema + FTS5), sessions.json index, JSONL transcripts, trajectory JSON | `agent/memory/`, `agent/session_db.py` |
| Scheduling Layer | Cron-style scheduled background work | `cron/` |
| Messaging Gateway | Platform-agnostic dispatch + per-adapter format/edit/truncate contract | `gateway/`, `gateway/adapters/*` |
| Machine Protocols | MCP server (FastMCP/stdio), ACP adapter (JSON-RPC stdio), TUI gateway JSON-RPC | `tools/mcp_*`, `acp_adapter/`, `acp_registry/`, `plugins/` |
| Delivery Surfaces (UI) | CLI shell, TUI, web dashboard SPA, web API, PTY WebSocket | `hermes_cli/`, `web/` |
| RL Environments | Training loops layered over the kernel | `environments/` |

Dependency direction: surfaces → protocols → gateway → kernel/tools → storage. Provider normalization sits beside the kernel. `[SI]`

## Feature Contract Table

22 rows covering the porting-relevant feature surface. Defect refs use D{pass}.{n} from mechanical (D1/D2/D6) and semantic (D3/D4/D5) reports. Action codes: **fbp** = fix before porting, **pd** = port differently, **lb** = leave behind. `[OF][SI]`

| # | Feature | Surface | Priority | Key Contracts | Defects | Action | Evidence |
|---|---|---|---|---|---|---|---|
| 1 | Agent turn-loop (compile prompt → call provider → parse tool calls → execute → persist) | Kernel | core | Prompt-cache stable across turns; tool result back-injection ordering | D5.6 (transform first-string-wins) | pd | `[OF]` |
| 2 | Provider call dispatch with retry + credential rotation | Kernel | core | Idempotent retry; classifier maps HTTP→retryable | D5.x (404→retryable indef.) | fbp | `[OF][PH]` |
| 3 | Tool registry execution + hooks (pre/post) | Kernel | core | Hooks fire on success AND dispatch failure | D5.4 (hooks not fired on dispatch failure) | fbp | `[OF]` |
| 4 | Tool approval / deny workflow (`/approve`, `/deny`) | CLI/TUI/Gateway | core | Default-deny outside allowlist; flags can override | D4.x allowlist-bypass under override | fbp | `[OF]` |
| 5 | Slash command core set (12 commands: /help /clear /model /save /resume /stop /new /retry /undo /compress /approve /deny) | CLI/TUI/Gateway | core | Reset triggers default `["/new","/reset"]` | — | pd | `[OF]` ctr-CF8 |
| 6 | Slash command important set (~20) | CLI/TUI | important | Stable command names | — | pd | `[OF]` |
| 7 | Slash command optional set (~26) | CLI | optional | — | — | lb | `[OF]` |
| 8 | Context compression / curator | Kernel | core | Cache-stable system prompt across compression boundary | prompt-cache invariant `[PH]` | fbp | `[SI]` |
| 9 | SessionDB persistence (SQLite v11 + FTS5) | Storage | core | schema_version=11; messages+sessions+state_meta+FTS5 | D2.x missing finally cleanup | fbp | `[OF]` |
| 10 | sessions.json atomic-replace index | Storage | core | Atomic write; legacy JSONL length-preferred read | D2.x | pd | `[OF]` |
| 11 | Legacy JSONL transcripts | Storage | important | Read path required, write path optional | — | pd | `[OF]` |
| 12 | Trajectory JSON export | Storage | optional | RL training format | — | lb | `[OF]` |
| 13 | MCP server (10 tools, FastMCP stdio, cursor-monotonic 200 ms poll) | Protocol | core | Cursor never goes backwards; mtime-poll default | prot-CF8 | pd | `[OF]` |
| 14 | ACP adapter (stdio JSON-RPC) | Protocol | core | JSON-RPC over stdio | — | pd | `[OF]` |
| 15 | TUI gateway JSON-RPC (27+ methods) | Protocol | important | Method names stable | D3.x | pd | `[OF]` |
| 16 | Multi-platform message dispatch (two-guard) | Gateway | core | Two-guard invariant prevents double-send | D3.1 (CRITICAL) | fbp | `[OF][PH]` |
| 17 | Per-adapter MAX_MESSAGE_LENGTH / format_message / truncate_message / REQUIRES_EDIT_FINALIZE | Gateway | core | Stable adapter contract | ctr-CF9+prot-CF7 | pd | `[OF]` |
| 18 | Telegram adapter | Gateway | core (MVP) | Bot token via .env | — | pd | `[OF]` |
| 19 | Discord adapter | Gateway | core (MVP) | Bot token via .env | — | pd | `[OF]` |
| 20 | Slack adapter | Gateway | important | OAuth refresh races `[PH]` | D2.x OAuth race | pd | `[OF]` |
| 21 | WhatsApp adapter | Gateway | important | Token + webhook | — | pd | `[OF]` |
| 22 | Matrix / Signal / Teams / Feishu / Yuanbao / Email / api_server / webhook adapters | Gateway | optional | Per-adapter token | — | lb (initial port) | `[OF]` |
| 23 | CLI surface (`hermes`) | UI | core | ANSI rendering; Unicode width | D6.6 ANSI escape leak | fbp | `[OF][PH]` |
| 24 | TUI surface (`hermes --tui`) | UI | important | ANSI semantics; resize | — | pd | `[OF][PH]` |
| 25 | Web dashboard SPA | UI | optional | WebSocket PTY + ANSI resize escape | — | lb (initial port) | `[OF]` |
| 26 | Web `/api/pty` WebSocket | UI | optional | ANSI resize escape protocol | — | lb (initial port) | `[OF][PH]` |
| 27 | Auth: GATEWAY_ALLOW_ALL_USERS / TEAMS_ALLOW_ALL_USERS env flips | Gateway/Web | important | One env var flips default-deny → default-allow | D4.x | fbp | `[OF][PH]` |
| 28 | Per-platform tokens via `~/.hermes/.env` | Config | core | env > .env > config.yaml > DEFAULT_CONFIG | D6.1 yaml→os.environ mutation (CRITICAL); D6.2 three loaders | fbp | `[OF]` |
| 29 | Three config loaders (load_cli_config, load_config, load_gateway_config) | Config | core | Consistency hazard | D6.2 | fbp | `[OF][PH]` |
| 30 | Plugin discovery (import side-effect at startup) | Kernel | important | Implicit import ordering | D1.x, D6.8 | fbp | `[OF][PH]` |
| 31 | Cron scheduling layer | Kernel | important | Background scheduling | — | pd | `[OF]` |
| 32 | RL environments | Kernel | optional | Trajectory JSON export | — | lb (initial port) | `[OF]` |
| 33 | Think-block tag filter (6 tags) | Kernel | core | Single filter primitive | prot-CF6 | pd | `[OF]` |
| 34 | Stream consumer sentinels (_DONE/_NEW_SEGMENT/_COMMENTARY) | Kernel | core | Sentinel ordering preserved | D3.x | fbp | `[OF]` |

## Protocol and State Notes

A port MUST preserve these invariants regardless of language: `[PH]`

- **Cursor-monotonic MCP**: the polling cursor never decreases across reconnects; reimplementations using FS-watch must still emit a monotonic cursor. Default = 200 ms mtime polling (per prot-CF8). `[OF]`
- **Atomic sessions.json**: write-to-temp + atomic-replace; partial files must never be visible to readers. `[OF]`
- **Prompt-cache stability**: the system prompt prefix must remain byte-identical across turns whenever provider-side prompt cache is desired. Compression must NOT mutate the prefix. `[PH]`
- **Two-guard message dispatch**: the gateway has two independent guards that together prevent double-send under retry; collapses arch-CF8+ctr-CF5+prot-CF1, surfaced as D3.1 (CRITICAL). A port that drops one guard will double-send. `[OF][PH]`
- **Stream consumer sentinel ordering**: `_NEW_SEGMENT` may precede content; `_DONE` must be last; `_COMMENTARY` is interleaved out-of-band. `[OF]`
- **Think-block tag set**: `<think>`, `<reasoning>`, `<REASONING_SCRATCHPAD>`, `<THINKING>`, `<thinking>`, `<thought>` — all six stripped by one filter primitive (per prot-CF6). `[OF]`
- **Schema_version=11 SessionDB**: tables `sessions`, `messages`, `state_meta`, plus FTS5 virtual table over message text. A port may choose its own DB but must preserve the *logical* schema. `[OF]`
- **Config precedence**: `env > .env file > config.yaml > DEFAULT_CONFIG`. D6.1 (yaml→os.environ mutation) BREAKS this precedence — a port must not reproduce the bug. `[OF][PH]`
- **Per-adapter contract**: every gateway adapter exposes `MAX_MESSAGE_LENGTH`, `format_message`, `truncate_message`, and `REQUIRES_EDIT_FINALIZE`. Codifying this as an interface is the porting unlock for ctr-CF9+prot-CF7. `[SI]`
- **Source-specific (NOT to preserve)**: import-side-effect plugin discovery (D1.x), three-config-loader duplication (D6.2), 167 KB auxiliary_client.py monolith (arch-CF10), `_last_resolved_tool_names` global. These should be redesigned, not ported. `[PH]`

## Portability Hazards

18 hazards consolidated from architecture, contracts, protocols, and both defect passes. `[PH]`

| # | Hazard | Source Phase | Impact | Mitigation |
|---|---|---|---|---|
| 1 | sync-loop-inside-async-gateway | architecture | Gateway calls `run_conversation` synchronously from async; blocks loop | Make kernel async-native in port |
| 2 | three-config-loaders coupling (D6.2) | mechanical | Three loaders read raw YAML; drift risk | Single config loader in port |
| 3 | yaml→os.environ mutation (D6.1, CRITICAL) | mechanical | Breaks env-var precedence | Never mutate os.environ from yaml |
| 4 | two-guard message dispatch (D3.1, CRITICAL) | semantic | Drop one guard → double-send | Codify as explicit dispatch state machine |
| 5 | ANSI escape leak (D6.6) + ANSI semantics | mechanical/protocols | Escapes leak into logs / non-tty | Centralize ANSI gate; respect `NO_COLOR` |
| 6 | Unicode width | protocols | Width-2 chars break TUI layout | wcwidth-equivalent library required |
| 7 | Shell quoting | architecture | Cross-shell quoting differs | Use exec-array, never string-shell |
| 8 | Process-global state mutation (`_last_resolved_tool_names` etc.) | mechanical | Test isolation broken; concurrent runs interfere | Replace globals with explicit context |
| 9 | Implicit import-side-effect ordering (D1.x, D6.8) | mechanical | Load order changes behavior; slow startup | Explicit registration in port |
| 10 | OAuth refresh races (Slack adapter) | semantic/D2.x | Concurrent refresh → invalid token | Single-flight refresh |
| 11 | Prompt-cache invariant | protocols | Compression mutating prefix invalidates provider cache | Append-only summary, never rewrite prefix |
| 12 | Think-block tag triple-duplication | protocols | Same regex written in 3 places | One filter primitive (prot-CF6) |
| 13 | FS locking on Windows | protocols | sessions.json atomic-replace assumes POSIX semantics | Windows path: use atomic-rename + retry |
| 14 | Env-var leak to subprocesses (D4.2) | semantic | `*_PROXY` not in subprocess blocklist | Explicit allowlist for subprocess env |
| 15 | Allowlist-bypass under override flags (D4.x) | semantic | Tool approval bypassed by flag combo | Re-derive default-deny in port |
| 16 | error_classifier 404→retryable indefinitely (D5.x) | semantic | Infinite retry loops | Treat 404 as terminal by default |
| 17 | transform_tool_result first-string-wins (D5.6) | semantic | Multiple transforms collide silently | Order-or-error policy |
| 18 | 167 KB auxiliary_client.py monolith (arch-CF10) | architecture | Untestable; defect concentration | Decompose into 5 modules in port |

## Defect Synthesis

**Total: 41 defects (3 critical, 17 high, 14 medium, 6 low)** across mechanical (22) + semantic (19) reports. Action mix: ~12 fix-before-porting, ~21 port-differently, ~5 leave-behind, 3 verify-during-impl. `[OF]`

Top 5 critical/high (must address in port design):

| Defect ID | Source Report | One-line Description | Severity | Porting Recommendation |
|-----------|---------------|----------------------|----------|------------------------|
| D6.1 | mechanical | yaml→os.environ mutation breaks env-var precedence | CRITICAL | fix before porting |
| D3.1 | semantic | two-guard message dispatch invariant (collapses arch-CF8+ctr-CF5+prot-CF1) | CRITICAL | fix before porting |
| D6.2 | mechanical | three-config-loaders consistency hazard | HIGH | port differently (single loader) |
| D5.4 | semantic | hooks not fired on dispatch failure | HIGH | fix before porting |
| D4.2 | semantic | env-var leak (`*_PROXY`) to subprocesses | HIGH | fix before porting |

Full synthesis (categorized counts and top items) is documented above; per-defect rows live in `defect-fix-tracker.md` for fix-before-porting items. `[OF]`

## Strategic Recommendations for Messaging Platforms (resolves arch-CF11 partially)

`[SI]` Tier this for the port:

- **Minimum viable (5):** CLI + MCP + ACP + Telegram + Discord. Delivers a usable agent on every developer machine + 2 chat platforms with simple bot-token auth.
- **Important (2):** Slack + WhatsApp. Slack adds OAuth complexity (D2.x race); WhatsApp adds webhook surface.
- **Optional / leave behind initial port (8):** Matrix, Signal, Teams, Feishu, Yuanbao, Email, generic api_server, generic webhook.
- **TUI + Web Dashboard:** TUI is *important* (replaces the CLI for many users); web dashboard SPA is *optional* (large surface area; ANSI-over-WebSocket PTY is a hazard).

Open question (arch-CF11 residual): which adapter surface gets the first non-Python adapter SDK? Recommend ACP because its JSON-RPC stdio shape is the smallest contract. `[OQ]`

## Resolution of Pre-Loaded Carry-Forwards

| CF ID | Status | Resolution |
|---|---|---|
| arch-CF10 | resolved | 167KB auxiliary_client.py decomposed (concept-level) into 5 modules: provider-call dispatch, retry policy, credential rotation, token accounting, error classification. Recorded in Layer Map row 2 + Hazard 18. |
| ctr-CF8 | resolved | 58 slash commands tiered: 12 core, ~20 important, ~26 optional/incidental. Feature table rows 5–7. |
| ctr-CF9 + prot-CF7 | resolved (collapsed) | Per-adapter contract codified: `MAX_MESSAGE_LENGTH`, `format_message`, `truncate_message`, `REQUIRES_EDIT_FINALIZE`. Feature table row 17 + Protocol notes. |
| prot-CF6 | resolved | Think-block tag list consolidated to one filter primitive over 6 tags. Feature table row 33 + Protocol notes. |
| prot-CF8 | resolved | MCP event bridge default = mtime-polling 200 ms (preserves cursor-monotonic). FS-watch deferred. Feature table row 13 + Protocol notes. |
| dss-CF1 | deferred-to-impl | verify by reading `model_tools.py` during reimplementation-spec phase. `[OQ]` |
| dss-CF2 | deferred-to-impl | verify by reading transform-result handlers during reimplementation-spec phase. `[OQ]` |
| dss-CF3 | deferred-to-impl | verify by reading `subprocess.Popen` call sites during reimplementation-spec phase. `[OQ]` |
| arch-CF11 | partially-resolved | Messaging platform tiers above. Residual open question: first non-Python adapter SDK target. |

## Observed Facts vs. Inferred Structure

### Observed Facts

- 11 top-level packages, ~200K LOC, Python 3.11+. `[OF]`
- 58 slash commands enumerated in `hermes_cli/commands.py:COMMAND_REGISTRY`. `[OF]`
- 18+ messaging-platform adapters under `gateway/adapters/`. `[OF]`
- 6 user-facing surfaces (CLI, TUI, web UI, API/SDK, gateway, storage/export). `[OF]`
- SessionDB schema_version=11; tables sessions/messages/state_meta + FTS5. `[OF]`
- Config precedence: env > .env > config.yaml > DEFAULT_CONFIG. `[OF]`
- Three config loaders. `[OF]`
- 41 total defects (22 mechanical + 19 semantic), 3 critical, 17 high. `[OF]`
- MCP server: 10 tools, stdio FastMCP, cursor-monotonic, 200 ms poll. `[OF]`
- Reset triggers default `["/new", "/reset"]`. `[OF]`

### Inferred Structure

- Layered composition: portable kernel + adapter shells (delivery surfaces). `[SI]`
- A port can ship a viable subset (CLI+MCP+ACP+Telegram+Discord) without re-implementing all 18+ adapters or the web SPA. `[SI]`
- The hardest port hazards are *invariants*, not algorithms (cursor-monotonic, atomic-replace, prompt-cache, two-guard, sync-in-async). `[SI]`
- The 167 KB auxiliary_client.py is a defect concentrator; decomposition is a pre-port refactor. `[SI]`
- Three config loaders are duplication, not necessity — one loader suffices. `[SI]`

## Domain Glossary

| Term | Definition | Where Used |
|---|---|---|
| Agent loop / turn-loop | The repeated cycle: build prompt → call provider → parse tool calls → execute → persist | `agent/` |
| Auxiliary client | The provider-normalization layer (167KB monolith) wrapping all model providers | `agent/auxiliary_client.py` |
| Curator | Context-compression component that summarizes/drops older turns | `agent/memory/` |
| Two-guard dispatch | The double-check pattern in gateway message send that prevents double-delivery | `gateway/` |
| Cursor-monotonic | MCP server polling invariant: the cursor only ever moves forward | `tools/mcp_*` |
| Prompt cache | Provider-side cache keyed by system-prompt prefix; requires byte-stable prefix | provider adapters |
| Sentinel | Stream control token (`_DONE`, `_NEW_SEGMENT`, `_COMMENTARY`) | stream consumer |
| Adapter contract | The 4-method interface every gateway platform implements | `gateway/adapters/*` |
| SessionDB | SQLite v11 store with FTS5 over message text | `agent/session_db.py` |
| Trajectory | RL training export format | `environments/` |
| Skill | Reusable agent capability bundle (30+ categories) | `skills/` |
| ACP | Agent Client Protocol; JSON-RPC over stdio | `acp_adapter/`, `acp_registry/` |
| MCP | Model Context Protocol; stdio FastMCP | `tools/mcp_*` |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| port-OQ1 | open-question | Which adapter SDK gets the first non-Python implementation? | Strategic decision; needs maintainer ruling |
| port-OQ2 | open-question | Is FS-watch ever a viable replacement for mtime-poll on MCP cursor? | Needs runtime test on each target OS |
| port-OQ3 | open-question | Can the web dashboard PTY WebSocket be replaced with a thinner protocol? | Needs UX evaluation |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| port-CF1 | reimplementation-spec | Pin opinionated decisions for the 12 core slash commands | Spec phase rubric |
| port-CF2 | reimplementation-spec | Pin async-native kernel decision (eliminate sync-in-async) | Spec phase rubric |
| port-CF3 | reimplementation-spec | Pin single-loader config decision (close D6.2) | Spec phase rubric |
| port-CF4 | reimplementation-spec | Pin auxiliary_client decomposition (5 modules) | Spec phase rubric |
| port-CF5 | reimplementation-spec | Pin two-guard dispatch as explicit state machine (close D3.1) | Spec phase rubric |
| port-CF6 | reimplementation-spec | Pin SessionDB logical schema (allow non-SQLite backends) | Spec phase rubric |
| port-CF7 | reimplementation-spec | Pin the per-adapter contract as a typed interface | Spec phase rubric |
| port-CF8 | reimplementation-spec | Resolve dss-CF1/2/3 via targeted reads (model_tools, transform handlers, subprocess sites) | Implementation evidence |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system summary, layer map, contract table, protocol notes, and porting findings are synthesized. | PASS | §System Summary; §Layer Map With Ownership (9 layers); §Feature Contract Table (34 rows); §Protocol and State Notes (10 invariants); §Strategic Recommendations + §Resolution of Pre-Loaded Carry-Forwards. |
| 2 | Portability hazards and open questions are separated from facts. | PASS | §Portability Hazards (18 rows) distinct from §Observed Facts vs. Inferred Structure; §Open Questions table separates port-OQ1/2/3 from facts. Inline `[OF]`/`[SI]`/`[PH]`/`[OQ]` tags throughout. |
| 3 | Feature importance is sorted for porting. | PASS | §Feature Contract Table column "Priority" = core/important/optional/incidental on every row; §Strategic Recommendations tiers messaging platforms; secondary appends tier each domain. |
| 4 | Defect Synthesis consolidates mechanical-defects.md and semantic-defects.md with porting recommendations (fix before porting / port differently / leave behind). | PASS | §Defect Synthesis: 41 defects (22 mechanical + 19 semantic), action mix ~12/~21/~5/3 verify-during-impl, top-5 table with explicit recommendations; full fbp subset in `defect-fix-tracker.md`. |
| 5 | Findings are marked with evidence levels. | PASS | Inline `[OF]`/`[SI]`/`[PH]`/`[OQ]` tags applied throughout the document; legend stated at top. |

**Validated by:** 2026-05-06 (porting phase, retry session)
**Overall:** PASS WITH GAPS

Gaps: port-OQ1 (first non-Python adapter SDK target) is a strategic decision requiring maintainer input; port-OQ2 (FS-watch viability) needs OS-specific runtime tests; port-OQ3 (PTY WebSocket replacement) needs UX evaluation. None are blockers for the reimplementation-spec phase: all major design decisions are queued in port-CF1..CF8 for resolution there.
