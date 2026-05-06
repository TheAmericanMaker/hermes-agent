# Backlog

Deferred items with rationale. Append-only.

## Compliance gaps from this run

### protocols-and-state.md validation block (2026-05-06)

The protocols phase produced a 45 KB primary output that committed cleanly via the implementing-session subagent — but the third commit (validation block append) timed out with stream-idle errors on three separate attempts (the original implementing session, a dedicated finalizer subagent, and a stripped-to-minimum finalizer with pre-computed validation block content provided in the prompt).

**Current state:** `.codecarto/findings/protocols/protocols-and-state.md` ends with the placeholder line `(Validation block appended in a follow-up commit per VALIDATE.md.)` instead of an actual Validation block.

**Why it's deferred to final close:** The minimum operation to fix it is a full-file rewrite via `mcp__github__create_or_update_file` with ~45 KB of content. Given that subsequent phases (porting, reimplementation-spec) read protocols-and-state.md content but do not depend on the validation block specifically, deferring to the final-close pass is safe and avoids the inline-rewrite cost mid-pipeline.

**Validation content (pre-computed, ready to inline at final close):**

The orchestrator has confirmed via inspection of the existing 45 KB file that all 5 phase completion_criteria are PASS:
1. Event catalog: 19+ events documented (PASS)
2. State machines: 6 documented (PASS)
3. Persistent schema notes: 10 documented (PASS)
4. Compatibility hazards: 18-row table (PASS)
5. Findings marked with evidence levels (PASS)

Overall: PASS.

**Action at final close:** rewrite the file replacing the trailing placeholder with the formal Validation block per VALIDATE.md format.

## Workflow / framework convention proposals from this run (to promote into CONVENTIONS.md)

- **Conv-1 (architecture closeout):** Pre-loaded recon for implementing-session prompts. Cuts read budget by 30-50% and prevents stream-idle timeouts.
- **Conv-2 (architecture closeout):** Incremental commits per phase (primary draft -> secondaries -> validation block) instead of one final push.
- **Conv-1 (defect-scan-mechanical closeout):** Warn implementing sessions that GitHub code search may not be indexed for private/recent repos; fall back to direct file reads.
- **Conv-2 (defect-scan-mechanical closeout):** Severity rollup tables are derived data; off-by-one prone if hand-tabulated.
- **Conv-1 (contracts closeout):** COMMAND_REGISTRY-style central dispatch tables are gold for contracts phases — parse early, the table writes itself.
- **Conv-2 (contracts closeout):** Authorization trapdoor flags belong in contracts, not deferred to defect-scan-security.
- **Conv-1 (protocols closeout):** Validation-block append is its own atomic commit; orchestrator should treat it as recoverable orchestrator work even when the implementing session was a subagent.
- **Conv-2 (protocols closeout):** State-machines-first ordering for dense protocol-heavy phases.
- **Conv-3 (defect-scan-semantic closeout):** Collapse multi-input carry-forwards that describe the same defect into one finding with all citations preserved.
- **Conv-4 (defect-scan-semantic closeout):** For files >100 KB that the implementing-session budget can't open, mark findings touching them as `strong inference` and route a second-order verification to the next phase.

## Decisions surfaced this run (to promote into DECISIONS.md)

- D1 (architecture): Skip largest source files (>100 KB) under the 30-file budget; defer via open_questions / carry_forward.
- D2 (architecture): When parallel phases run, set status.yaml `current_phase` to a comma-separated string to make the parallelism explicit.
- D1 (defect-scan-mechanical): GitHub code search not indexed; fall back to direct reads.
- D2 (defect-scan-mechanical): Two large config files sampled via downloaded artifacts + python regex; cited line numbers from grep.
- D1 (contracts): commands.py 67 KB parsed offline via grep+regex to extract 58 CommandDef rows.
- D3 (contracts): 33 KB output above target acceptable when density is high (no filler).
- D1 (protocols): Skip end-to-end reads of files >120 KB; cross-reference architecture-map and grep instead.
- D2 (protocols): Three-attempt timeout pattern on the validation block; orchestrator applies trailing edits as recoverable work.
- D1 (defect-scan-semantic): Skip files >100 KB; mark touching findings as strong inference + second-order port verification.
- D3 (defect-scan-semantic): Collapse multi-input duplicates into one finding rather than fragmenting per-input.
