# Decisions

Numbered log of cross-cutting decisions diverging from spec or making strategic calls. Append-only.

## D1. Use the deep-audit pipeline variant

**Date:** 2026-05-06
**Phase:** planning
**Decision:** Run `pipeline-full-with-deep-audit.yaml` (7 phases, with split mechanical/semantic defect scan) rather than `pipeline-full-with-audit.yaml` (6 phases, single early defect scan) or `pipeline.yaml` (5 phases, no defect scan).
**Rationale:** User asked for "the full deep run." Hermes-agent at ~200K LOC has enough complexity that the split mechanical/semantic defect scan provides materially different views: mechanical is run early (light context, structural defects), semantic runs late (full contracts/protocols context, contract drift). One critical finding (D3.1 two-guard) actually collapsed three carry-forwards from three upstream phases — only possible with the late-semantic split.

## D2. Strategic Alignment Hook resolved as language-agnostic

**Date:** 2026-05-06
**Phase:** planning (pre-Phase-7)
**Decision:** Phase 7 (reimplementation-spec) uses `templates/reimplementation-spec.md` (default, language-agnostic) NOT `templates/reimplementation-spec-opinionated.md`.
**Rationale:** User selection via AskUserQuestion. Goal: spec describes contracts, protocols, and modules in stack-neutral way; any team can port to Rust, Go, TypeScript, etc.

## D3. Findings committed to the analysis branch

**Date:** 2026-05-06
**Phase:** framework bootstrap
**Decision:** Override `.codecarto/.gitignore` to allow `findings/` and `closeouts/` to be committed to `claude/codecarto-hermes-analysis-abvQm`. Keep `scratch/` ignored.
**Rationale:** User selection via AskUserQuestion. The branch is the deliverable; un-committed findings would leave nothing to review.

## D4. Phases 2-4 ran in parallel after architecture

**Date:** 2026-05-06
**Phase:** orchestration
**Decision:** Phases 2 (defect-scan-mechanical), 3 (contracts), and 4 (protocols) launched simultaneously after Phase 1 (architecture) completed. Orchestrator serialized status.yaml writes after each parallel return.
**Rationale:** Each depends only on architecture. Sequential would have been ~3x slower. The framework's concurrent-write hazard on status.yaml was avoided by routing all status updates through the orchestrator (subagents return content; orchestrator commits state).

## D5. Disjoint secondary ownership during parallel execution

**Date:** 2026-05-06
**Phase:** Phases 3+4 parallel run
**Decision:** Phase 3 (contracts) owned `public-surfaces` and `config-model` secondary appends; Phase 4 (protocols) owned `runtime-lifecycle` and `state-and-storage`. No file overlap.
**Rationale:** Both phases natively wanted to append to all four. Disjoint ownership eliminated concurrent-write conflicts on the secondary catalogs without serializing the phases.

## D6. Carry-forward duplicates collapsed in defect-scan-semantic

**Date:** 2026-05-06
**Phase:** Phase 5 (defect-scan-semantic)
**Decision:** When three carry-forwards (arch-CF8, ctr-CF5, prot-CF1) all described the same two-guard message dispatch invariant, the phase collapsed them into one finding (D3.1) rather than producing three separate findings.
**Rationale:** Triple-citation more accurately reflects the recurring-mistake-surface than three siblings. Now codified as Convention C9.

## D7. 167 KB auxiliary_client.py decomposition is a pre-port refactor, not port-time

**Date:** 2026-05-06
**Phase:** Phase 6 (porting)
**Decision:** The reimplementation-spec treats decomposing `auxiliary_client.py` (167 KB) as work that should happen in the source repo BEFORE the port begins, not during. Recommended split into 5 sub-modules.
**Rationale:** The monolith is not the final form regardless of target language. Cleaning it in Python first reduces churn during the cross-language port and isolates "refactor" from "port" in the test/review process. Now codified as Convention C11.

## D8. Protocols-and-state.md validation block deferred to BACKLOG

**Date:** 2026-05-06
**Phase:** Phase 4 (protocols) finalization
**Decision:** After three subagent timeouts attempting to append the validation block to the 45 KB primary output, the orchestrator marked the file's content PASS (verified by inspection) but logged the formal validation block append as a deferred BACKLOG item rather than blocking phase progression.
**Rationale:** Subsequent phases (porting, reimpl-spec) read protocols-and-state.md for content, not validation. Deferring to final close avoided a 45 KB inline rewrite mid-pipeline at a moment when both ahead phases were viable.

## D9. Tier recommendations from porting carried into reimpl-spec

**Date:** 2026-05-06
**Phase:** Phase 7 (reimplementation-spec)
**Decision:** The porting phase's tier recommendations (5 core / 2 important / 8 optional messaging platforms; 5/4/4 surfaces) were adopted verbatim by the reimpl-spec's three scope tiers (minimum-viable / major-workflow-parity / full-parity).
**Rationale:** The porting bundle made the case based on integration cost; the reimpl-spec accepted it because the architecture-map analysis already showed that 18+ messaging platforms are independently portable, so the tier cut is mechanical rather than design-driven.

## D10. Default-deny auth posture in the reimpl-spec

**Date:** 2026-05-06
**Phase:** Phase 7 (reimplementation-spec)
**Decision:** The spec recommends DEFAULT-DENY auth across all surfaces, with explicit allow-list flags preserved as opt-in (not default-flip).
**Rationale:** Hermes-agent's `GATEWAY_ALLOW_ALL_USERS` / `TEAMS_ALLOW_ALL_USERS` env-var override is a one-flag trapdoor that flips default-deny to default-allow. Reimpl-spec inverts this: default-deny is the only safe posture for a multi-platform agent that may run as a service.

## D11. Reset policy in UTC + per-user TZ

**Date:** 2026-05-06
**Phase:** Phase 7 (reimplementation-spec)
**Decision:** The spec recommends UTC + per-user TZ override for the reset policy `at_hour` field, NOT local-time-of-the-process as in the source.
**Rationale:** Hermes-agent's reset policy `at_hour` is evaluated against `datetime.now()` (process local time). For multi-user gateway deployments this gives different reset times per user depending on where the process happens to run. UTC + per-user TZ preserves the user-visible behavior while making the implementation timezone-correct.

## D12. Linear session chains with explicit branch operation

**Date:** 2026-05-06
**Phase:** Phase 7 (reimplementation-spec)
**Decision:** The spec keeps the `parent_session_id` linear chain model from `hermes_state.SessionDB` schema_version=11 (compression splits chain linearly via `parent_session_id`), but adds an explicit branch operation for future fork semantics.
**Rationale:** Linear chains are simpler to reason about and migrate. Branch model is a future-only optimization with no current use case in hermes-agent. Explicit branch op preserves the option without paying the complexity cost upfront.
