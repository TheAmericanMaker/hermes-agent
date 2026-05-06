# Closeout - 2026-05-06 - defect-scan-mechanical

**Phase:** defect-scan-mechanical (Phase 2 of pipeline-full-with-deep-audit)
**Project:** hermes-agent
**Branch:** claude/codecarto-hermes-analysis-abvQm
**Session role:** implementing

## What was done

Ran passes 1, 2, 6 (logic & correctness, error handling, config & environment) over hermes-agent. Resolved three architecture carry-forwards (arch-CF5/CF6/CF7) with concrete findings. Routed six new items (concurrency, security, contract drift) forward to defect-scan-semantic.

Committed in two increments: primary draft, then validation block append.

## Outputs

- `findings/defect-scan-mechanical/mechanical-defects.md` - 27,058 bytes (with validation block)

## Validation

Four PASS, one PARTIAL (criterion 4: count rollup table is off by one - high=8 stated, 9 actual; non-blocking, every finding is detailed in the body). Overall: PASS WITH GAPS.

## Key findings (top-of-mind)

- D6.1: yaml -> os.environ mutation in gateway/config.py breaks env-var precedence (critical, fix before porting).
- D2.1: single broad `except` clause in gateway/config.py swallows all per-platform mapping errors (high, fix before porting).
- D6.2: three-config-loaders coupling cluster - hermes_cli.config.load_config / read_raw_config + gateway.config.load_gateway_config + inline yaml.safe_load reads, each with independent caching.
- 22 defects total: 2 critical, 9 high, 7 medium, 4 low. ~9 fix-before-porting, ~10 port-differently, ~3 leave-behind.

## Decisions Beyond Prompt

- **D1**: GitHub native code search returned 0 hits (repo not indexed); fell back to direct `get_file_contents` reads. Stayed at 8 source-file reads (well under 30-cap) by relying on AGENTS.md and architecture-map signals.
- **D2**: Two large config files (hermes_cli/config.py 199KB, gateway/config.py 76KB) sampled via downloaded artifacts + python regex extraction; line numbers cited from grep output rather than verified end-to-end.

## Proposed Conventions

- **Conv-1**: "GitHub code search may not be indexed for private/recent repos." Implementing-session prompts should warn agents to fall back to direct file reads when search returns empty, rather than assuming a clean codebase.
- **Conv-2**: "Severity rollup tables are derived data and should be auto-generated." Manual rollups are off-by-one prone; ports of the framework could add a tooling step.

## Citations

- Primary output: `.codecarto/findings/defect-scan-mechanical/mechanical-defects.md`
- Pipeline: `.codecarto/workflow/pipeline-full-with-deep-audit.yaml` (defect-scan-mechanical phase)
- Open questions: dsm-OQ1
- Carry-forward: dsm-CF1 through dsm-CF6 (all to defect-scan-semantic)
