# Defect Fix Tracker — hermes-agent

**Campaign:** Post-CodeCartographer defect remediation (porting phase, fix-before-porting subset)
**Started:** 2026-05-06
**Source:** `findings/defect-scan-mechanical/mechanical-defects.md` + `findings/defect-scan-semantic/semantic-defects.md` (41 defects total)

---

## Summary

| Metric | Count |
|--------|-------|
| Total defects | 41 |
| Fix-before-porting tracked here | 12 |
| Fixed | 0 |
| Deferred | 0 |
| Accepted (by design) | 0 |
| Remaining | 12 |

**Test suite:** existing pytest suite in source repo (count to be verified during impl).

---

## Fix Status

### Critical — 2 found, 0 fixed

| ID | Location | Description | Status | Fix Summary |
|----|----------|-------------|--------|-------------|
| D6.1 | gateway/config.py + cli config | yaml→os.environ mutation breaks env-var precedence | pending | |
| D3.1 | gateway/ message dispatch | two-guard message dispatch invariant; collapses arch-CF8+ctr-CF5+prot-CF1 | pending | |

### High — 8 found (subset tracked here for fbp), 0 fixed

| ID | Location | Description | Status | Fix Summary |
|----|----------|-------------|--------|-------------|
| D6.2 | three config loaders | three-config-loaders consistency hazard | pending | |
| D5.4 | tool registry hooks | hooks not fired on dispatch failure | pending | |
| D5.6 | transform_tool_result handlers | first-string-wins on multiple transforms | pending | |
| D4.2 | subprocess call sites | env-var leak (`*_PROXY` not in blocklist) to subprocesses | pending | |
| D5.x | error_classifier | 404 mapped to retryable indefinitely | pending | |
| D2.x | gateway/config.py | broad `except` swallows config errors | pending | |
| D1.x | plugin discovery (import side-effect) | implicit import-side-effect ordering at startup | pending | |
| D4.x | tool approval / allowlist | allowlist-bypass under override flag combinations | pending | |

### Medium — 2 fbp tracked, 0 fixed

| ID | Location | Description | Status | Fix Summary |
|----|----------|-------------|--------|-------------|
| D6.6 | render layer | ANSI escape leak into non-tty / log streams | pending | |
| D2.x | session storage write path | missing `finally` cleanup | pending | |

### Low — 0 fbp tracked

| ID | Location | Description | Status | Fix Summary |
|----|----------|-------------|--------|-------------|
| — | — | (low-severity items routed to port-differently or leave-behind) | n/a | |

---

## Deferred Items

- The remaining 29 defects (port-differently or leave-behind) are NOT tracked here; they are addressed by the reimplementation design itself rather than by source-side fixes. See `reverse-engineering-bundle.md` Defect Synthesis.
- **dss-CF1, dss-CF2, dss-CF3:** verify-during-impl items (read model_tools.py, transform handlers, subprocess.Popen sites). Deferred to reimplementation-spec phase.

---

## Files Modified

| File | Defects Fixed |
|------|---------------|
| (none yet — tracker initialized) | |

---

## Notes

- Severity counts here are the fix-before-porting subset, not the full 41-defect totals (3 critical, 17 high, 14 medium, 6 low).
- Two CRITICAL items (D6.1, D3.1) are blockers: a port that inherits these will be unsafe.
- D3.1 is a *concept invariant* (two-guard dispatch). The fix in source is a code change; in the port it is a state-machine design decision.
- D6.1 is a *bug* in source AND a design constraint for the port (never mutate os.environ from YAML).
