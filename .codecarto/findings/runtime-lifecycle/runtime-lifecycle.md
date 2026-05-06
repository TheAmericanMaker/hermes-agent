## 2026-05-06 — porting phase

**Lifecycle behaviors tiered for the port:**

- **core:** agent turn-loop (compile prompt → call provider → parse tool calls → execute → persist), provider-call dispatch with retry + credential rotation, tool registry execution + hooks (pre/post fire on success AND dispatch failure — closes D5.4), context compression that preserves prompt-cache prefix stability, stream consumer sentinel ordering (`_NEW_SEGMENT` first, `_DONE` last). Rationale: these are the system; without them there is no agent.
- **important:** cron scheduling layer, gateway message dispatch with two-guard invariant (closes D3.1 CRITICAL), per-adapter format/edit/truncate contract, plugin discovery (replace import-side-effect with explicit registration to fix D1.x/D6.8). Rationale: required for parity on multi-platform workflows and reliable startup.
- **optional:** RL training loops, dashboard PTY lifecycle, `/api/pty` resize-escape protocol. Rationale: layered atop the kernel; can be deferred without affecting core agent behavior.
- **incidental:** sync-loop-inside-async-gateway pattern (`run_conversation` called sync from async). Rationale: a Python-specific accident; the port should be async-native end-to-end (port-CF2).
