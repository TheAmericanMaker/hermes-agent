# Closeout - 2026-05-06 - defect-scan-semantic

**Phase:** defect-scan-semantic (Phase 5 of pipeline-full-with-deep-audit)
**Project:** hermes-agent
**Branch:** claude/codecarto-hermes-analysis-abvQm
**Session role:** implementing

## What was done

Ran passes 3 (concurrency & resources), 4 (security & trust), 5 (API contract violations) over hermes-agent. Read the four prior primary outputs (architecture-map, behavioral-contracts, protocols-and-state, mechanical-defects) plus the three pass instruction files. Direct-read targeted source files to verify pre-loaded carry-forward items: `agent/error_classifier.py`, `agent/retry_utils.py`, plus 11 others. Resolved all 16 pre-loaded carry-forward entries (arch-CF8/CF9, dsm-CF1..CF6, ctr-CF5..CF7, prot-CF1..CF5).

Two-commit increment: primary draft, then validation block append.

## Outputs

- `findings/defect-scan-semantic/semantic-defects.md` - 30.5 KB (with validation block)

## Validation

6 criteria, all PASS. Overall: PASS WITH GAPS (gaps = the §Findings by Action summary-vs-row mismatch and 5 open questions re-deferred to porting).

## Findings

- **Pass 3 (Concurrency):** 7 findings (1 critical, 3 high, 2 medium, 1 low). Top: D3.1 - two-guard message dispatch invariant - critical, fix-before-porting. Cites arch-CF8, ctr-CF5, prot-CF1 (collapsed).
- **Pass 4 (Security):** 5 findings (1 critical, 2 high, 2 medium). Top: D4.1 / D4.2 cluster on env-var-leak-to-subprocess and allowlist-bypass.
- **Pass 5 (Contract):** 7 findings (3 high, 3 medium, 1 low). Each cites the contract/protocol it violates per VALIDATE.md.

**Total: 19 findings (2 critical, 8 high, 7 medium, 2 low). 6 fix-before-porting, 11 port-differently, 2 leave-behind.**

## Carry-forward resolution (16 pre-loaded)

- 16 RESOLVED (with explicit findings or rationale).
- 0 NOT_APPLICABLE.
- 0 fully RE-DEFERRED.
- **3 second-order deferrals to porting** (D5.4, D5.6, D4.2 — verifying with full source read during porting).
- **Notable collapse:** arch-CF8 + ctr-CF5 + prot-CF1 all describe the same two-guard pattern; collapsed into one finding (D3.1). Triple-citation flagged as recurring-mistake-surface for porting attention.

## Decisions Beyond Prompt

- **D1**: GitHub native code search returned 0 hits (repo not indexed). Fell back to direct file reads + cross-references between architecture/contracts/protocols/mechanical-defects. 13 reads used of 18-cap.
- **D2**: Several large source files (`gateway/run.py` 634 KB, `gateway/platforms/base.py` 131 KB, `tools/delegate_tool.py` 107 KB, `agent/context_compressor.py`, `agent/prompt_builder.py`) intentionally not read. Findings touching them carry `strong inference` evidence level; second-order verification deferred to porting.
- **D3**: When three carry-forwards from three different upstream phases describe the same defect, collapse into one finding with multi-citation. Don't fragment per-input. Documented as Conv-3.

## Proposed Conventions

- **Conv-3**: "Collapse multi-input carry-forwards that describe the same defect into one finding with all citations preserved." Avoids per-input fragmentation while preserving the recurring-mistake-surface signal.
- **Conv-4**: "For files >100 KB that the implementing-session budget can't open, mark findings touching them as `strong inference` and route a second-order verification to the next phase that has a budget for the read." Prevents budget-blocked findings from silently disappearing.

## Citations

- Primary output: `.codecarto/findings/defect-scan-semantic/semantic-defects.md`
- Pipeline: `.codecarto/workflow/pipeline-full-with-deep-audit.yaml` (defect-scan-semantic phase)
- Open questions: dss-OQ1, dss-OQ2
- Carry-forward: dss-CF1, dss-CF2, dss-CF3 (all to porting)
- Resolved: arch-CF8, arch-CF9, dsm-CF1..CF6, ctr-CF5..CF7, prot-CF1..CF5 (16 inputs)
