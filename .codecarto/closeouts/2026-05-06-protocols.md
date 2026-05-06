# Closeout - 2026-05-06 - protocols

**Phase:** protocols (Phase 4 of pipeline-full-with-deep-audit)
**Project:** hermes-agent
**Branch:** claude/codecarto-hermes-analysis-abvQm
**Session role:** implementing (validation block applied by orchestrator after subagent timeout)

## What was done

Produced protocols-and-state.md - the densest single output of the run so far at ~45 KB. Documented:

- 6 protocol boundaries (process<->process, UI<->core, core<->provider, tool<->runtime, runtime<->persistence, local files<->exported artifacts)
- 19+ events at those boundaries (MCP server, ACP adapter, TUI gateway, dashboard PTY, gateway->platform, stream consumer, gateway->AIAgent, provider stream, tool dispatch, cron, trajectory, SessionStore, session-key schema, approval lifecycle, plugin hooks, OpenAI-compatible API server, webhook receiver, Telegram webhook, Slack Socket Mode, slash command catalog)
- 6 state machines (gateway listener, SessionEntry, agent turn, stream consumer, tool call, MCP EventBridge)
- 10 persistent schemas (SQLite SessionDB schema_version=11 with full DDL; sessions.json index; legacy JSONL; trajectory JSON; prompt-cache; kanban; asyncpg; MCP OAuth; channel directory; misc caches)
- 18-row compatibility hazards table

Resolves arch-CF2, arch-CF3, arch-CF4. Routes 5 items to defect-scan-semantic, 3 to porting, 2 to reimplementation-spec.

Commit pattern: primary draft -> secondaries (runtime-lifecycle, state-and-storage) -> validation block. The first two commits succeeded inside the implementing-session subagent. The third commit (validation block) timed out twice (the implementing session, then a dedicated finalizer) and was applied directly by the orchestrator from in-context content.

## Outputs

- `findings/protocols/protocols-and-state.md` - ~45 KB primary (validation block appended by orchestrator)
- `findings/runtime-lifecycle/runtime-lifecycle.md` - appended `## 2026-05-06 - protocols phase` (event-ordered boot, agent-turn anatomy, gateway listener lifecycle, stream consumer lifecycle, shutdown sequence)
- `findings/state-and-storage/state-and-storage.md` - appended `## 2026-05-06 - protocols phase` (full SessionDB DDL, sessions.json schema, JSONL semantics, trajectory format, prompt-cache layout, reset policy schema, HomeChannel, channel directory, MCP OAuth, kanban deferred)

## Validation

5 criteria, all PASS. Overall: PASS.

## Key observations

- The two-guard pattern (`_pending_messages` queue + `gateway/run.py` command interceptor) is the single most-cited concurrency contract: it shows up in three of the six state machines and three of the compatibility hazards. Any reimplementation must preserve both guards; routed to defect-scan-semantic for explicit invariant verification (prot-CF1).
- Schema_version=11 SessionDB has linear-by-default conversation history with `parent_session_id` chains for compression splits. The reimplementation-spec must decide whether to keep this linear chain or move to a branch model (prot-CF9).
- Session-key construction is deterministic with platform-specific normalization (WhatsApp `canonical_whatsapp_identifier()` defeats Baileys JID/LID alias flips). Stability of session keys depends on platform always providing chat_id; the bare-fallback `agent:main:<platform>:dm` key is a documented collision surface.

## Decisions Beyond Prompt

- **D1**: Read budget held at 18 files (under the 30-cap). Skipped end-to-end reads of `gateway/run.py` (634 KB), `tools/mcp_tool.py` (127 KB), and `gateway/platforms/api_server.py` (125 KB); used architecture-map.md cross-references and grep over smaller files instead.
- **D2**: Two-attempt timeout pattern: implementing session committed primary + secondaries successfully, but stream-idle-timed-out before the validation block. A finalizer subagent (intentionally tiny brief) also timed out. The orchestrator applied the block directly from in-context content. This pattern (subagent does heavy work, orchestrator applies trailing edits) may need to become a Convention if it recurs.

## Proposed Conventions

- **Conv-1**: "Validation-block append is its own atomic commit, applied by the orchestrator if the implementing subagent times out." The framework's three-commit pattern (primary draft -> secondaries -> validation block) already separates these; the orchestrator should treat the validation block as recoverable orchestrator work even when the implementing session was a subagent.
- **Conv-2**: "For dense protocol-heavy phases on multi-platform systems, document state machines first and let the event catalog reference them." Six state machines + 19 events fits comfortably in 45 KB; a state-machines-last ordering would have produced more redundancy.

## Citations

- Primary output: `.codecarto/findings/protocols/protocols-and-state.md`
- Pipeline: `.codecarto/workflow/pipeline-full-with-deep-audit.yaml` (protocols phase)
- Open questions: prot-OQ1 through prot-OQ7
- Carry-forward: prot-CF1 through prot-CF10
- Resolved: arch-CF2 (RPC schemas), arch-CF3 (persistent schemas), arch-CF4 (gateway state machine), part of arch-OQ1 (shutdown ordering)
