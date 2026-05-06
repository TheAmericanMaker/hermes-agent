# Reimplementation Spec — hermes-agent

> **Scope:** Language-agnostic reimplementation plan for `hermes-agent`. This document is the terminal artifact of CodeCartographer Phase 7. It treats the source repository as evidence, not as a template. A port may be written in any language/runtime; module boundaries and file names below are conceptual, not source-mirrored.

> **Evidence classification used throughout:**
> - `OF` = observed fact (cited in prior phase findings)
> - `SI` = strong inference (derived from multiple observed facts)
> - `PH` = portability hazard (must be designed around in the port)
> - `OQ` = open question (deferred to spike list)

---

## System Summary

`hermes-agent` is, conceptually, a **persistent multi-surface conversational agent runtime**: a long-lived process that maintains durable conversation sessions, brokers between one or more LLM providers, executes a registered tool catalog with policy hooks, and exposes the same agent through many delivery surfaces (terminal, MCP server, ACP adapter, chat platforms, web dashboard). [SI]

A port should restate the system as four concerns layered on top of one another:

1. **Core agent semantics** — a deterministic turn loop that accumulates messages, invokes a provider abstraction, dispatches tool calls, and persists session state. The shape of a "turn" and the prompt-cache stability invariant are the load-bearing concepts; everything else is plumbing. [OF]
2. **Provider and tool plumbing** — a normalized provider interface (one shape regardless of upstream SDK), a tool registry with explicit registration and pre/post hooks, and a credential/token accounting layer. [OF]
3. **Messaging and delivery surfaces** — adapters that translate platform-specific events (Telegram update, Discord message, MCP request, ACP message, HTTP request, terminal keystroke) into core-agent turns and translate agent output back. A messaging gateway sits in front of platform adapters and applies a security/flood-control policy. [OF]
4. **Persistence and orchestration** — a session database (relational, append-only event log of turns plus indexed session metadata), a scheduler for cron-driven prompts, and a skills/plugins runtime. [OF]

The port's success criterion is **behavioral parity on the acceptance scenarios in §Acceptance Scenarios**, not structural fidelity to the Python source. The port is encouraged to collapse Python-specific helper layers (e.g., the source's `auxiliary_client.py` mega-module) into idiomatic abstractions in the target language.

---

## Conceptual Module Model

The system decomposes into **eleven** conceptual modules. None of these names should be taken as filenames; they are roles.

### 1. Agent Kernel

| Field | Value |
|---|---|
| **Responsibility** | Run the turn loop: accept a user message, accumulate context, invoke the provider, dispatch any tool calls, append results, repeat until terminal. |
| **Public inputs** | `(session_id, user_message, attachments?, control_flags?)` |
| **Public outputs** | Stream of `assistant_chunk` events, terminal `assistant_message`, `tool_call` and `tool_result` events, persisted session updates. |
| **Owned state** | In-memory turn buffer; current cancellation token; turn sequence counter. |
| **Invariants** | (a) Prompt cache prefix is stable within a session unless an explicit reset/branch occurs; (b) every emitted `tool_call` is followed by exactly one `tool_result` before the next provider call; (c) turns are appended monotonically. |
| **Collaborators** | Provider Normalization Layer, Tool Registry, Session Store, Token Accounting. |

### 2. Provider Normalization Layer

| Field | Value |
|---|---|
| **Responsibility** | Present a single internal "chat completion" shape regardless of which upstream LLM provider is in use. |
| **Public inputs** | `(messages[], tools[], model_id, sampling_params, stream?)` |
| **Public outputs** | Either a streaming iterator of normalized chunks, or a single normalized completion with usage metadata. |
| **Owned state** | None per call; shared retry policy config; shared credential rotator. |
| **Invariants** | (a) Tool-call shape is identical across providers; (b) usage tokens are reported in a single canonical schema; (c) errors are classified into a fixed taxonomy before being raised. |
| **Collaborators** | Provider Adapters (one per vendor), Retry Policy, Credential Rotator, Error Classifier. |

### 3. Tool Registry

| Field | Value |
|---|---|
| **Responsibility** | Hold the catalog of callable tools, their JSON-schema input contracts, and their pre/post hooks. Resolve tool-call requests to executions. |
| **Public inputs** | `register(tool_descriptor)`, `dispatch(tool_call) -> tool_result`. |
| **Public outputs** | Normalized `tool_result` payloads; structured errors for unknown / unauthorized / malformed calls. |
| **Owned state** | The tool table (name → descriptor); per-tool authorization policy. |
| **Invariants** | (a) Registration is **explicit** — no import-time side effects register a tool; (b) every dispatch passes through pre-hooks before execution and post-hooks before returning; (c) tool names are unique. |
| **Collaborators** | Authorization Policy, Skills Runtime, individual tool implementations. |

### 4. Session Store

| Field | Value |
|---|---|
| **Responsibility** | Durably persist sessions: turns, metadata, parentage, and reset markers. |
| **Public inputs** | `create_session`, `append_turn`, `read_session`, `branch_session`, `set_reset_marker`. |
| **Public outputs** | Session records and turn streams. |
| **Owned state** | Session database (relational, schema_version pinned); session index file; transcript files (legacy interop). |
| **Invariants** | (a) Schema version is pinned and migrations are forward-only; (b) session index updates use atomic-replace semantics (write temp + rename); (c) turn ordinals are monotonic per session; (d) `parent_session_id` chains are linear by default. |
| **Collaborators** | Migration Runner, Index Writer, Trajectory Exporter. |

### 5. Messaging Gateway

| Field | Value |
|---|---|
| **Responsibility** | Front-door for all platform-originated messages: apply identity check, authorization, flood control, and command parsing, then route to either a control handler or the Agent Kernel. |
| **Public inputs** | `inbound_message(platform_id, user_id, text, attachments?)`. |
| **Public outputs** | Either a control-command response, an Agent Kernel invocation, or a rejection with reason. |
| **Owned state** | Per-user rate-limit state; identity cache. |
| **Invariants** | (a) **Default-deny**: unknown principals are rejected before any agent work; (b) "always-reach" control commands (e.g., `/help`, `/auth`) bypass the flood guard and the auth guard in a single, audited code path; (c) every inbound message produces exactly one outbound acknowledgement (success, rejection, or error). |
| **Collaborators** | Authorization Policy, Platform Adapters, Agent Kernel. |

### 6. Platform Adapters

| Field | Value |
|---|---|
| **Responsibility** | Translate one vendor's wire events (Telegram update, Discord message, Slack event, MCP RPC, ACP message, HTTP request) into Messaging Gateway calls and translate agent outputs back into vendor-shaped responses. |
| **Public inputs** | Vendor-specific event objects. |
| **Public outputs** | Vendor-specific response objects; delivery receipts. |
| **Owned state** | Per-platform connection / webhook state; per-platform message-id mapping. |
| **Invariants** | (a) An adapter never bypasses the Messaging Gateway; (b) MCP cursors emitted by the MCP adapter are monotonically increasing per session. |
| **Collaborators** | Messaging Gateway, vendor SDKs (where wrapped). |

### 7. Delivery Surfaces (CLI / TUI / Web)

| Field | Value |
|---|---|
| **Responsibility** | Provide first-party human-facing UIs that drive the Agent Kernel directly (CLI/TUI) or via HTTP (web dashboard). |
| **Public inputs** | Keystrokes, command-line arguments, HTTP requests. |
| **Public outputs** | Terminal renders, HTTP responses, websocket streams. |
| **Owned state** | Render buffer; HTTP session cookies (if any); websocket subscriptions. |
| **Invariants** | (a) The CLI never bypasses the Session Store — every CLI turn is persisted; (b) the web dashboard authenticates every request before mutating state. |
| **Collaborators** | Agent Kernel, Session Store, Authorization Policy. |

### 8. Scheduler

| Field | Value |
|---|---|
| **Responsibility** | Trigger cron-style prompts and route results to a configured delivery target. |
| **Public inputs** | `schedule(spec, prompt_template, delivery_target)`. |
| **Public outputs** | Scheduled invocations of the Agent Kernel and forwarding of results. |
| **Owned state** | Job table; last-fired timestamps; per-user reset clocks. |
| **Invariants** | (a) Reset clocks are stored in **UTC** with an optional per-user TZ offset; (b) a missed window does not trigger backfill unless explicitly configured; (c) job dispatch is idempotent within one window. |
| **Collaborators** | Agent Kernel, Messaging Gateway, Configuration Loader. |

### 9. Authorization Policy

| Field | Value |
|---|---|
| **Responsibility** | Decide whether a principal may invoke a surface, a tool, or a control command. |
| **Public inputs** | `authorize(principal, action, resource)`. |
| **Public outputs** | `allow` / `deny(reason)`. |
| **Owned state** | Allow-list, role table, per-tool ACLs. |
| **Invariants** | (a) **Default-deny** — unknown principal/action pairs return `deny`; (b) the policy is consulted by the Messaging Gateway, the Tool Registry, and the Web Surface; (c) decisions are logged. |
| **Collaborators** | Messaging Gateway, Tool Registry, Delivery Surfaces. |

### 10. Configuration Loader

| Field | Value |
|---|---|
| **Responsibility** | Resolve runtime configuration from a precedence chain: defaults < config file < environment variables < CLI flags. |
| **Public inputs** | Config file path, `os.environ`, argv. |
| **Public outputs** | An immutable `Config` value passed to all modules at startup. |
| **Owned state** | None at runtime (one-shot resolution). |
| **Invariants** | (a) **The loader does NOT mutate the process environment**; (b) precedence is honored exactly; (c) secrets are not echoed in logs. |
| **Collaborators** | Every module that reads config (essentially all of them). |

### 11. Skills Runtime

| Field | Value |
|---|---|
| **Responsibility** | Discover and execute file-based skills (sub-prompts and subagent recipes) from a configured home directory. |
| **Public inputs** | Skill name, invocation parameters. |
| **Public outputs** | Skill execution result (a tool-style payload). |
| **Owned state** | Skill index (path → manifest). |
| **Invariants** | (a) Skills are loaded from a single configured root; (b) skill execution is mediated by the Tool Registry's hooks; (c) skill names are namespaced. |
| **Collaborators** | Tool Registry, Configuration Loader. |

### Layer Split

| Module | Layer | Notes |
|---|---|---|
| Agent Kernel | core semantics | Must survive the port behaviorally unchanged. |
| Provider Normalization Layer | core semantics | Internal shape is core; vendor adapters under it are adapters. |
| Tool Registry | core semantics | Registration is part of the semantic contract. |
| Session Store | core semantics | Schema and atomic-replace are semantic invariants. |
| Authorization Policy | core semantics | Default-deny is a semantic invariant. |
| Configuration Loader | core semantics | Precedence is a semantic invariant. |
| Provider Adapters (per vendor) | adapters | Wrap or replace per dependency stance. |
| Platform Adapters (per platform) | adapters | Wrap per vendor SDK. |
| Messaging Gateway | adapters | Sits between adapters and core. |
| Scheduler | adapters | Replaces `croniter` with native facilities. |
| Delivery Surfaces (CLI / TUI / Web) | delivery surfaces | Replace UI libraries with idiomatic target equivalents. |
| Skills Runtime | delivery surfaces | File-based; portable as-is. |

---

## Required Behaviors

The port MUST exhibit each of the following. IDs `RB-1..RB-22` will be cited from §Acceptance Scenarios.

- **RB-1** A `(session_id, user_message)` tuple submitted to the Agent Kernel returns either a streaming response or a single response containing the assistant text and any tool-call/tool-result pairs in order. [OF]
- **RB-2** Within a single session, the prompt prefix sent to the provider is byte-stable across consecutive turns absent an explicit reset or branch — i.e., prompt-cache continuity holds. [OF, PH]
- **RB-3** Every tool call emitted by the provider produces exactly one tool result before the next provider call. [OF]
- **RB-4** Tools are registered explicitly at startup; importing a tool module does not auto-register it. [OF]
- **RB-5** Pre-hooks run before tool execution; post-hooks run before the result is returned to the kernel. [OF]
- **RB-6** Unknown / unauthorized / malformed tool calls return a structured error result, not an exception thrown to the provider. [OF]
- **RB-7** Sessions persist across process restart; restarting the process and replaying `read_session` yields the prior turns in order. [OF]
- **RB-8** Session-index updates are atomic from a reader's perspective (no partial-write states). [OF]
- **RB-9** `parent_session_id` chains are linear by default; "branch" is an explicit operation that creates a new session whose `parent_session_id` points to the source turn. [SI]
- **RB-10** Reset markers are stored in UTC; conversion to a user's local time is a presentation-layer concern. [SI, PH]
- **RB-11** The Messaging Gateway rejects any inbound message from an unknown principal with a structured rejection (default-deny). [SI]
- **RB-12** "Always-reach" control commands (e.g., `/help`, `/auth`) reach their handler even when the principal is unknown OR when flood control would otherwise rate-limit them — both guards are bypassed in one path. [OF, PH]
- **RB-13** Flood control is per-principal and resets on a fixed window; the window is documented and configurable. [OF]
- **RB-14** Every inbound message produces exactly one outbound acknowledgement (success, rejection, or error). [SI]
- **RB-15** MCP cursors are monotonically increasing per session and survive reconnection. [OF]
- **RB-16** ACP, MCP, and chat-platform adapters all funnel through the same Messaging Gateway entrypoint. [OF]
- **RB-17** The Provider Normalization Layer reports usage tokens in a canonical `{prompt, completion, total}` shape regardless of vendor. [OF]
- **RB-18** Provider errors are classified into `{retryable_transient, retryable_rate_limit, fatal_auth, fatal_model_not_found, fatal_validation, unknown}`; HTTP 404 from the provider classifies as `fatal_model_not_found` (NOT retryable). [PH — fixes D5.x]
- **RB-19** Configuration precedence is `defaults < config_file < environment < CLI`; no module reads config except the Configuration Loader. [OF]
- **RB-20** The Configuration Loader does NOT write to `os.environ` or any global mutable state. [PH — fixes D6.1]
- **RB-21** Cron schedules fire within one scheduling window of their target time; missed windows do NOT auto-backfill unless explicitly configured. [SI]
- **RB-22** The CLI surface persists every turn through the Session Store before rendering output. [OF]

---

## Protocols and Persisted State

The port MUST preserve these wire and on-disk invariants. Schemas may be re-encoded (JSON vs CBOR vs protobuf) so long as semantics survive.

### Wire formats

- **MCP**: JSON-RPC 2.0 over stdio or stream. Cursor field on list-style endpoints is monotonic per session and opaque to the client. [OF]
- **ACP**: Adapter exposes a message channel; each ACP message round-trips a single agent turn. [OF]
- **Chat platforms (Telegram / Discord / Slack / WhatsApp)**: vendor-native event shapes; the gateway normalizes them into `(platform_id, user_id, text, attachments)`. [OF]
- **Web dashboard**: HTTP/JSON; websocket for streaming chunks. [OF]
- **Provider call shape (internal)**: `{messages: [{role, content, tool_calls?, tool_call_id?}], tools: [...], model, sampling}` — a single shape across vendors. [OF]
- **Tool result shape**: `{tool_call_id, status: "ok"|"error", content}`. [OF]

### Persisted state

- **Session database**: relational. Schema version is pinned (currently `schema_version=11` in source; the port should pick a starting version and document migrations). Tables (conceptual): `sessions`, `turns`, `tool_calls`, `attachments`. [OF]
- **Sessions index file**: a separate index (the source uses `sessions.json`) updated via **write-temp-and-rename atomic-replace**. Readers must tolerate either the old or new content but never a torn write. [OF]
- **Legacy transcripts**: JSONL, one record per turn, append-only. The port may treat these as export-only. [OF]
- **Trajectory JSON**: per-session export consumed by training pipelines. The port MAY defer this to Tier 3 (full parity). [SI]

### State machine — session lifecycle

`created → active → (compressed | branched | reset)*  → archived`. A `reset` clears the prompt prefix at a marker and starts a new prefix; a `branch` forks a new session whose `parent_session_id` references the branch point. Compression is conceptually a form of branch. [SI]

### Prompt cache invariant

Within an `active` session, the byte sequence `messages[0..N]` sent to the provider on turn `N+1` is identical to that sent on turn `N` plus the new user/tool messages. Any change to a prior message — including reordering, format changes, or whitespace edits — invalidates the cache and is a defect unless preceded by an explicit reset/branch. [OF, PH]

---

## External Dependencies

Stances inherited from the porting bundle (Phase 6) and validated against the conceptual modules above:

| Dependency | Stance | Rationale |
|---|---|---|
| `openai` SDK | replace | Native HTTP + the canonical chat-completions shape is small; the SDK adds little value cross-language. |
| `anthropic` SDK | wrap | Vendor's streaming and tool-use shape benefits from a thin facade. |
| `mistralai` SDK | wrap | Same as Anthropic. |
| `boto3` (Bedrock) | wrap | Vendor SDK encapsulates SigV4; wrap rather than reimplement. |
| `prompt_toolkit` | replace | Most target languages have idiomatic line-editing libraries. |
| `rich` | replace | Use the target language's standard rendering. |
| `aiosqlite` | replace | Use native SQLite bindings of the target language. |
| `asyncpg` | replace | Use native Postgres bindings (Tier 2/3). |
| `pydantic` | replace | Use the target language's idiomatic schema/validation. |
| `jinja2` | replace | Use a target-native templating library. |
| `PyJWT` | replace | JWT is small; native crypto in target. |
| `python-telegram-bot` | wrap | Per-platform facade. |
| `discord.py` | wrap | Per-platform facade. |
| `slack-bolt` | wrap | Per-platform facade. |
| `croniter` | replace | Cron parsing is small; replace with native. |
| `playwright` | postpone | Subagent-only; defer to Tier 3. |
| `firecrawl-py` | postpone | Subagent-only; defer to Tier 3. |

---

## Portability Hazards

Numbered hazards `PH-1..PH-15`. Each is a risk specific to porting, with mitigation guidance.

- **PH-1: Prompt-cache byte stability.** Re-encoding the prompt (different JSON serializer, different float formatting, sorted-keys vs insertion-order) silently breaks the cache and inflates cost. **Mitigation:** lock a canonical serializer in the Provider Normalization Layer; add a regression check that re-serializing a stored prompt is byte-identical.
- **PH-2: Atomic-replace semantics for the session index.** Some target runtimes (notably Windows and some FUSE filesystems) do not provide POSIX rename atomicity. **Mitigation:** abstract `atomic_write(path, bytes)` and document the platform support matrix; use platform-native atomic primitives.
- **PH-3: Two-guard message dispatch.** The current Python source applies the auth guard and the flood guard in two places, and "always-reach" commands must skip both — a known defect (D3.1). **Mitigation:** the port should funnel ALL inbound messages through one guard pipeline; "always-reach" is a property of the command descriptor, not a code path.
- **PH-4: 404 misclassification.** The current Python source retries on HTTP 404 indefinitely (D5.x). **Mitigation:** error classifier is exhaustive over a fixed taxonomy; 404 = `fatal_model_not_found`; the test suite includes a 404-from-provider scenario.
- **PH-5: yaml→env mutation.** Source mutates `os.environ` from config (D6.1), breaking precedence. **Mitigation:** Configuration Loader returns an immutable Config value; no module reads `os.environ` after loader runs.
- **PH-6: Streaming back-pressure.** The provider streams chunks faster than some delivery surfaces (Telegram message-edit rate limit) can render. **Mitigation:** delivery surfaces buffer and coalesce; the kernel does not wait on surface rendering.
- **PH-7: Concurrent session writes.** Two surfaces writing to the same session ID simultaneously (e.g., user typing in CLI while a cron fires) can interleave turns. **Mitigation:** Session Store enforces per-session write serialization (advisory lock or transaction).
- **PH-8: Tool execution is unbounded.** A tool that opens a network connection or shells out can run arbitrarily long. **Mitigation:** Tool Registry enforces a per-tool timeout and surfaces it as a structured error.
- **PH-9: Credential rotation across providers.** A single rotation event must invalidate all in-flight provider clients. **Mitigation:** provider clients are stateless and resolved per call from a `CredentialResolver` rather than constructed at startup.
- **PH-10: MCP cursor monotonicity across reconnects.** A reconnect that resets the cursor causes duplicates. **Mitigation:** persist cursors per (session, endpoint) tuple.
- **PH-11: Schema migrations on cold start.** Migrations that run on startup race with first request. **Mitigation:** migrations complete before the gateway opens its listening sockets.
- **PH-12: Skill discovery on filesystem.** Symlinks, case-sensitivity, and large directories can introduce surprises. **Mitigation:** Skills Runtime indexes lazily and validates manifests; errors are non-fatal.
- **PH-13: Time zones for reset markers.** Storing reset markers in local time leaks the host's TZ into session data. **Mitigation:** UTC at rest; per-user TZ offset on read.
- **PH-14: Provider token-accounting drift.** Different vendors count tokens differently; a unified usage shape must document its source. **Mitigation:** the Provider Normalization Layer records both `vendor_reported` and a `normalized` token count.
- **PH-15: Default-allow regressions.** Adding a new tool or a new control command may default-allow if the policy is consulted only at the Gateway. **Mitigation:** Authorization Policy is consulted at every enforcement point (Gateway, Tool Registry, Web Surface).

---

## Implementation Sequence

Build in the order below. Each milestone produces a runnable artifact.

### Scope Tiers

**Minimum viable port** (the "agent talks, persists, and is reachable"):
1. Configuration Loader (no env mutation; precedence honored).
2. Session Store with relational backend, schema_version pinned, atomic-replace index.
3. Provider Normalization Layer with one provider adapter (Anthropic recommended due to first-class tool-use shape).
4. Tool Registry with explicit registration + pre/post hooks.
5. Agent Kernel (turn loop, prompt-cache stability).
6. Authorization Policy (default-deny).
7. Messaging Gateway (single-guard pipeline).
8. CLI delivery surface.
9. MCP adapter.
10. ACP adapter.
11. Telegram adapter.
12. Discord adapter.

Exit criteria: acceptance scenarios `AS-1..AS-7` pass.

**Major-workflow parity** (covers primary daily use):
13. Slack adapter.
14. WhatsApp adapter.
15. Web dashboard (HTTP + websocket streaming).
16. Scheduler with cron parsing and per-user TZ.
17. Branch operation (explicit) and compression.
18. Two additional provider adapters (OpenAI native HTTP; one of Mistral / Bedrock).
19. Token accounting with normalized usage shape.

Exit criteria: acceptance scenarios `AS-1..AS-12` pass.

**Full parity** (everything the original does):
20. Remaining 11+ messaging platforms.
21. Skills Runtime with file-based registry under a configured home.
22. RL training pipeline (trajectory export consumer).
23. OAuth refresh flows for vendor SDKs.
24. Postgres backend mode for Session Store.
25. Subagent-only tools (browser automation, web crawling) — `playwright`/`firecrawl-py` postponed dependencies.

Exit criteria: full behavioral parity; opt-in feature flags for legacy JSONL transcripts.

---

## Acceptance Scenarios

Black-box, concrete I/O. Each scenario cites the contract IDs from Phase 3 (`CTR-#`) where applicable and the required-behavior IDs above.

| # | Scenario | Input | Expected Output / Side Effect |
|---|----------|-------|-------------------------------|
| AS-1 | Single-turn assistant response (RB-1, CTR-1) | New session; user sends `"What is 2+2?"` | Assistant returns a text response containing `"4"`. One row appended to `turns` for the user message and one for the assistant message. |
| AS-2 | Tool call round-trip (RB-3, RB-5, CTR-3) | New session with a `now()` tool registered; user sends `"What time is it in UTC?"` | Provider emits one `tool_call` for `now`; pre-hook runs; tool returns ISO-8601 UTC; post-hook runs; provider's next response includes that timestamp; persisted turns include both `tool_call` and `tool_result` records in order. |
| AS-3 | Prompt-cache continuity (RB-2, PH-1) | Same session; turn 1 then turn 2 | Bytes of `messages[0..N]` sent to the provider on turn 2 are byte-identical to those sent on turn 1. |
| AS-4 | Session restart (RB-7, CTR-5) | Submit one turn; restart the process; read the session | All turns appear in the same order they were appended. |
| AS-5 | Atomic index update (RB-8) | Concurrent reader polls the session index while a writer appends a new session | Reader sees either the pre-write or post-write state; never a torn JSON. |
| AS-6 | Default-deny rejection (RB-11, CTR-7) | Unknown principal sends a non-control message via Telegram | Gateway returns a structured rejection; no kernel invocation; one outbound acknowledgement only. |
| AS-7 | Always-reach control command (RB-12, CTR-8) | Unknown principal sends `/help` via Telegram | Help text returned; auth guard and flood guard both bypassed; rejection NOT issued. |
| AS-8 | 404 from provider is fatal (RB-18, PH-4) | Configure an invalid `model_id`; submit a turn | Error classifies as `fatal_model_not_found`; NO retry; user gets a structured error. |
| AS-9 | Config precedence honored (RB-19, RB-20) | `model_id` set in config file as `A`, in env as `B`, on CLI as `C`; start the agent | Resolved model is `C`; `os.environ` for `model_id` is unchanged after startup. |
| AS-10 | MCP cursor monotonicity (RB-15) | Connect MCP client; list resources; disconnect; reconnect; list again with prior cursor | Server returns items strictly after the prior cursor; no duplicates. |
| AS-11 | Branch is explicit (RB-9) | On an active session, invoke `branch(turn_id=T)`; submit a new turn on the branch | New session created with `parent_session_id=T_session` and `parent_turn_id=T`; original session unchanged. |
| AS-12 | Cron fires in the right window (RB-21) | Schedule `0 9 * * *` for a user with TZ offset `-05:00`; advance clock to UTC 14:00 on the next day | One Agent Kernel invocation occurs within one window; result is delivered to the configured platform. |
| AS-13 | Flood control resets per window (RB-13) | Send `N+1` messages where `N` is the per-window cap | `N+1`-th message is rejected with a flood-control reason; after the window elapses, a new message is accepted. |
| AS-14 | Tool unauthorized (RB-6, PH-15) | Provider emits a `tool_call` for a tool the principal is not authorized for | Tool Registry returns a structured `error` result of `unauthorized`; provider sees the structured error, not an exception. |

---

## Deliberate Non-Goals

- **Mirroring source filenames** (`auxiliary_client.py`, etc.) — the port should choose idiomatic names.
- **JSONL legacy transcripts as the primary store** — relational Session Store is canonical; JSONL is export-only.
- **Auto-backfill of missed cron windows** — explicit opt-in only.
- **Browser-automation and web-crawling subagent tools at MVP** — `playwright` / `firecrawl-py` postponed to Tier 3.
- **Postgres mode at MVP** — SQLite-equivalent only at Tier 1.
- **Reproducing Python `prompt_toolkit` keybindings** — use the target language's idiomatic line editor.
- **Reproducing the `rich` rendering layer** — use idiomatic terminal rendering.
- **RL training pipeline at MVP** — Tier 3.
- **Skills runtime at MVP** — Tier 3 (skill execution is an extension; core works without it).

---

## Defect Handling (mandatory)

Per pipeline rule, every defect from Phase 2 + Phase 5 + Phase 6's tracker is either **designed-around** or **left-behind** in this spec. Below covers all 12 fix-before-porting entries plus the 3 critical defects called out in the porting bundle.

| Defect ID | Class | Status | Treatment |
|---|---|---|---|
| D6.1 | yaml→`os.environ` mutation breaks env precedence | **designed-around** | RB-19 + RB-20 + Configuration Loader §invariants forbid mutation; Acceptance AS-9 verifies. |
| D3.1 | Two-guard message dispatch (always-reach commands must bypass BOTH) | **designed-around** | Messaging Gateway §invariants mandate single-guard pipeline; RB-12; PH-3 mitigation; AS-7 verifies. |
| D5.x (404→retryable) | Error classifier classifies HTTP 404 as retryable indefinitely | **designed-around** | Provider Normalization Layer §invariants pin a fixed taxonomy with 404 = `fatal_model_not_found`; RB-18; AS-8 verifies. |
| D-mech-01 | Tool registration via import side-effects | **designed-around** | Tool Registry §invariants require explicit registration; RB-4. |
| D-mech-02 | Session index torn-write window | **designed-around** | Session Store §invariants require atomic-replace; PH-2; AS-5 verifies. |
| D-mech-03 | Tool execution lacks timeout | **designed-around** | PH-8 mitigation: per-tool timeout in Tool Registry. |
| D-mech-04 | Concurrent session writes interleave | **designed-around** | PH-7 mitigation: per-session write serialization. |
| D-sem-01 | Credential rotation does not invalidate in-flight clients | **designed-around** | PH-9 mitigation: clients resolved per call from `CredentialResolver`. |
| D-sem-02 | MCP cursor reset on reconnect | **designed-around** | PH-10 mitigation: cursors persisted per (session, endpoint); RB-15; AS-10 verifies. |
| D-sem-03 | Token accounting drift across providers | **designed-around** | PH-14 mitigation: normalized usage shape with both vendor and normalized counts; RB-17. |
| D-sem-04 | Schema migrations race first request | **designed-around** | PH-11 mitigation: migrations complete before listeners open. |
| D-sem-05 | Skill discovery surprises (symlinks/case) | **designed-around** | PH-12 mitigation: lazy index + non-fatal manifest errors. |
| D-sem-06 | Reset markers stored in local time | **designed-around** | PH-13 mitigation; RB-10; UTC at rest with per-user TZ on read. |
| D-port-01 | Default-allow regressions when adding new tools | **designed-around** | PH-15 mitigation: Authorization Policy consulted at every enforcement point; AS-14 verifies. |
| D-port-02 | Streaming back-pressure on Telegram message-edit limit | **designed-around** | PH-6 mitigation: delivery surfaces buffer/coalesce. |
| D-port-03 | Source-language helper layers (`auxiliary_client.py`) | **left-behind** | Explicitly collapsed: the port refactors this into Provider Normalization Layer + Retry Policy + Credential Rotator + Token Accounting + Error Classifier as separate modules. |
| D-port-04 | Python-typing-driven module splits | **left-behind** | Idiomatic target-language packaging; not preserved. |
| D-port-05 | Legacy JSONL primary-store path | **left-behind** | Demoted to export-only; canonical store is relational. |

---

## Carry-Forward Resolutions (from prior phases)

Items routed to this phase that have been resolved here:

- **arch-CF11** (Platform MVP cut) — resolved: Tier 1 lists the 5 core platforms (CLI, MCP, ACP, Telegram, Discord); Tier 2 adds Slack + WhatsApp; remainder Tier 3.
- **ctr-CF10** (12 acceptance scenarios baseline) — resolved: §Acceptance Scenarios contains 14 black-box scenarios.
- **ctr-CF11** (default-deny vs default-allow) — resolved: **default-deny** mandated in Authorization Policy §invariants and RB-11.
- **prot-CF9** (linear vs branched parent_session_id) — resolved: **linear by default; branch is explicit** (RB-9, AS-11).
- **prot-CF10** (reset policy at_hour local vs UTC) — resolved: **UTC at rest with per-user TZ override on read** (RB-10, PH-13).
- **port-OQ1** (ACP-first non-Python adapter recommendation) — resolved: ACP is in Tier 1; recommended as the first adapter to validate the Messaging Gateway with a non-chat, non-CLI surface.

---

## Known Unknowns

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| KU-1 | spike | Canonical prompt-serializer choice for byte-stable cache (JSON sorted-keys vs insertion-order) | Needs measurement against each target provider's tokenizer; depends on target language's JSON library. |
| KU-2 | spike | Atomic-replace primitive on Windows / FUSE filesystems for the session index | Needs runtime test on each supported OS. |
| KU-3 | spike | Streaming chunk coalescing strategy for Telegram message-edit rate limit | Needs empirical test against Telegram's current limits. |
| KU-4 | maintainer-decision | Default model and provider for MVP | Cost / availability tradeoff; pipeline cannot decide. |
| KU-5 | maintainer-decision | Whether to support both stdio and HTTP transports for MCP at Tier 1 | Depends on consumer fleet. |
| KU-6 | spike | Postgres mode write-fanout and per-session locking | Tier 3; needs load test. |
| KU-7 | spec-ruling | Should the Skills Runtime accept skills outside the configured home (e.g., absolute paths)? | Security policy decision. |
| KU-8 | spike | Subagent isolation model (thread / process / sandbox) for tools that shell out | Platform-sensitive. |
| KU-9 | maintainer-decision | Branch / compression operation: user-visible name, default trigger threshold | UX decision. |
| KU-10 | spike | Per-platform identity binding (e.g., Telegram user_id vs internal principal) | Needs design pass during Tier 1 implementation of each adapter. |

## Carry-Forward (post-pipeline)

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| RIS-CF1 | spike | Validate prompt-cache byte-stability across at least two providers | Requires running code; cannot be closed in spec phase. |
| RIS-CF2 | spike | Validate atomic-replace on each target OS | Requires runtime test. |
| RIS-CF3 | delta | Reconcile this spec with any changes to source between Phase 6 and port start | Standard spec drift handling. |

## Spike List

- **Prompt-cache byte stability spike** (KU-1, RIS-CF1): build a test harness that submits two consecutive turns and diffs the byte streams sent to the provider; gate Tier 1 exit on this passing.
- **Atomic-replace spike** (KU-2, RIS-CF2): run the Session Store's index writer concurrently with N readers on Linux, macOS, Windows, and a FUSE filesystem.
- **Streaming back-pressure spike** (KU-3, PH-6): saturate the chunk producer; measure per-platform delivery latency and edit-rate compliance.
- **Subagent isolation spike** (KU-8, PH-8): evaluate thread vs process vs sandbox tradeoffs for tool execution.
- **Per-platform identity binding spike** (KU-10): exercise Telegram, Discord, Slack, WhatsApp identity schemes against a unified principal model.
- **Postgres performance spike** (KU-6): Tier-3 write-fanout test with N concurrent sessions.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | Concept-level modules are defined. | PASS | §Conceptual Module Model: 11 concept-named modules (Agent Kernel, Provider Normalization Layer, Tool Registry, Session Store, Messaging Gateway, Platform Adapters, Delivery Surfaces, Scheduler, Authorization Policy, Configuration Loader, Skills Runtime), each with responsibility / inputs / outputs / state / invariants / collaborators; Layer Split assigns each to core / adapter / surface. |
| 2 | Required behaviors are stated. | PASS | §Required Behaviors lists RB-1..RB-22 with evidence tags; behaviors are derived from Phase 3 contracts and cross-referenced from acceptance scenarios. |
| 3 | Protocol and persisted state expectations are stated. | PASS | §Protocols and Persisted State documents wire formats (MCP, ACP, chat platforms, web, internal provider call shape, tool result shape), persisted state (relational session DB, atomic-replace sessions index, JSONL legacy, trajectory JSON), session-lifecycle state machine, and the prompt-cache byte-stability invariant. |
| 4 | Acceptance scenarios and known unknowns are included. | PASS | §Acceptance Scenarios (14 black-box scenarios AS-1..AS-14 with concrete inputs and observable outputs, citing RB and CTR IDs); §Known Unknowns (KU-1..KU-10) and §Spike List. |
| 5 | Defects identified in either scan are explicitly designed-around or noted as "left behind", with the choice cited. | PASS | §Defect Handling: 18 entries covering 3 critical defects (D6.1, D3.1, D5.x) + 12 fix-before-porting tracker items (D-mech-01..04, D-sem-01..06, D-port-01..02) + 3 left-behind items (D-port-03..05). Each cites the §invariant, RB, PH, or AS that addresses it. |
| 6 | Findings are marked with evidence levels. | PASS | Evidence legend defined at top (`OF`/`SI`/`PH`/`OQ`); applied throughout System Summary, Required Behaviors, Protocols and Persisted State, Portability Hazards, and the conceptual module invariants. |

**Validated by:** 2026-05-06 (reimplementation-spec phase, session `claude/codecarto-hermes-analysis-abvQm`)
**Overall:** PASS WITH GAPS

**Gap notes:** All criteria PASS. The PASS WITH GAPS classification reflects that 10 Known Unknowns (KU-1..KU-10) and 3 post-pipeline carry-forwards (RIS-CF1..3) remain open by design — they require runtime spikes or maintainer decisions that no later pipeline phase can close. These are tracked under §Known Unknowns and §Carry-Forward (post-pipeline). The phase is complete; downstream port work owns these spikes.
