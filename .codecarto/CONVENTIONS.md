# Conventions

Project-specific patterns curated by the orchestrator across implementing sessions. Append-only when generalization is confirmed across at least two sessions.

## Subagent / orchestrator conventions (from 2026-05-06 hermes-agent run)

### C1. Pre-load recon for implementing-session prompts

**Pattern:** When the orchestrator spawns an implementing session against a large codebase, the orchestrator should pass structural recon (entry points, dir layout, manifests, prior phase signal) directly in the prompt rather than asking the implementing session to re-discover.

**Evidence:** Confirmed in architecture closeout. Re-confirmed in porting closeout: a first attempt with explicit instructions to read prior-phase outputs timed out at ~8 minutes; a retry with 170 KB of prior-phase signal compressed into the prompt and a hard 6-file read budget completed cleanly in ~9 minutes including 3 commits.

**Cuts read budget by 30-50% and prevents stream-idle timeouts on slow-search phases.**

### C2. Incremental commits per phase

**Pattern:** Implementing sessions should push primary draft -> secondaries -> validation block as three separate commits rather than batching everything into one final push. Survives partial-write timeouts and matches the framework's read-write segregation (primary owns load-bearing claims, secondaries own enumerated detail).

**Evidence:** Architecture phase landed cleanly via incremental pattern despite a prior attempt timing out. Protocols phase confirmed: primary + secondaries committed cleanly; only the trailing validation block append timed out (and was recoverable).

### C3. Validation-block append is orchestrator-level recoverable work

**Pattern:** Treat the validation block as recoverable work even when the implementing session was a subagent. The framework's three-commit pattern already separates it from primary/secondary writes, so when an implementing-session subagent times out before appending the block, the orchestrator can either retry with a finalizer subagent OR apply the block directly from in-context content.

**Evidence:** Protocols phase. Three subagent attempts (original implementing, dedicated finalizer, minimal-finalizer-with-pre-computed-content) all timed out. Logged as a deferred BACKLOG item; planned for inline orchestrator rewrite at next session.

### C4. Disjoint secondary ownership for parallel phases

**Pattern:** When running two phases in parallel that both append to the same secondary catalogs, the orchestrator must split ownership so they touch disjoint files.

**Evidence:** Phases 3 (contracts) and 4 (protocols) both wanted to append to `public-surfaces`, `runtime-lifecycle`, `state-and-storage`, `config-model`. Orchestrator partitioned: contracts owned `public-surfaces` + `config-model`; protocols owned `runtime-lifecycle` + `state-and-storage`. No conflicts during parallel execution.

### C5. GitHub code search may not be indexed

**Pattern:** Implementing-session prompts should warn agents that `mcp__github__search_code` may return 0 hits for private/recent repos. When this happens, fall back to direct `mcp__github__get_file_contents` reads.

**Evidence:** Defect-scan-mechanical, defect-scan-semantic, and contracts phases all reported 0 search hits. Direct reads were the actual data source.

### C6. Severity rollups are derived data

**Pattern:** Counts in summary tables (e.g. "defects by severity") drift off-by-one when hand-tabulated. Either generate them mechanically or expect a PARTIAL on the corresponding validation criterion.

**Evidence:** Defect-scan-mechanical scored PARTIAL on count rollup criterion (high=8 stated, 9 actual). Defect-scan-semantic scored PASS WITH GAPS partly for similar mismatch.

### C7. COMMAND_REGISTRY-style central dispatch tables are gold for contracts phases

**Pattern:** When a target codebase has a single registry of commands/handlers, parse it into a table early — one regex extracts the entire surface and the contract rows largely write themselves.

**Evidence:** Contracts phase parsed `hermes_cli/commands.py:COMMAND_REGISTRY` (67 KB file) via grep+regex offline to extract 58 CommandDef rows in one shot.

### C8. Authorization trapdoor flags belong in contracts

**Pattern:** Default-allow vs default-deny is a contract surface, not a defect. When env vars or config flags can flip auth posture (e.g. `*_ALLOW_ALL_USERS`), document them in the contracts phase, not deferred to defect-scan-security.

**Evidence:** Contracts phase identified `GATEWAY_ALLOW_ALL_USERS` / `TEAMS_ALLOW_ALL_USERS` as one-flag trapdoors during the auth model section.

### C9. Collapse multi-input carry-forwards that describe the same defect

**Pattern:** When carry-forwards from three different upstream phases describe the same recurring-mistake-surface, collapse them into one finding with all citations preserved. Avoids per-input fragmentation while keeping the danger signal visible.

**Evidence:** Defect-scan-semantic D3.1 collapses arch-CF8 + ctr-CF5 + prot-CF1 (all describe the two-guard message dispatch invariant). Triple-citation flagged as recurring-mistake-surface for porting attention.

### C10. Files >100 KB get strong-inference + second-order verification

**Pattern:** When a source file is too large to open under the implementing-session budget, mark findings touching it as `strong inference` and route a second-order verification to the next phase that has budget for the read.

**Evidence:** Semantic-defects phase routed dss-CF1/CF2/CF3 to porting for `model_tools.py` / transform handlers / `subprocess.Popen` enumeration. Porting phase further deferred to impl-time read.

### C11. Pre-port refactor for monolithic source files

**Pattern:** For source files >150 KB that the port will need to decompose, recommend the decomposition as a pre-port refactor in the source repo. This separates "language port" from "structural cleanup" and reduces port risk.

**Evidence:** Porting phase recommended the 167 KB `auxiliary_client.py` be decomposed into 5 sub-modules (provider-call dispatch, retry policy, credential rotation, token accounting, error classification) BEFORE the port begins.

### C12. State-machines-first for protocol-heavy phases

**Pattern:** When documenting a protocol-heavy phase on a multi-platform system, define state machines first and let the event catalog reference them. State-machines-last produces redundancy.

**Evidence:** Protocols phase: 6 state machines + 19 events fit comfortably in 45 KB; the state machine references in the event catalog avoided per-event re-explanation of lifecycle.
