# Closeout - 2026-05-06 - contracts

**Phase:** contracts (Phase 3 of pipeline-full-with-deep-audit)
**Project:** hermes-agent
**Branch:** claude/codecarto-hermes-analysis-abvQm
**Session role:** implementing (retry; first attempt timed out without committing)

## What was done

Produced behavioral-contracts.md with 34 feature contracts (18 slash + 7 CLI subcommand + 7 API/SDK/Background + 2 storage), 12 black-box acceptance scenarios, security & authorization model, configuration model, special behaviors. Resolved arch-CF1 (slash-command catalog) by enumerating 58 commands from hermes_cli/commands.py COMMAND_REGISTRY into the public-surfaces secondary append.

Three-commit increment: primary draft -> secondaries (public-surfaces, config-model) -> validation block.

## Outputs

- `findings/contracts/behavioral-contracts.md` - 33.2 KB (with validation block)
- `findings/public-surfaces/public-surfaces.md` - +9 KB appended (`## 2026-05-06 - contracts phase`)
- `findings/config-model/config-model.md` - +7 KB appended

## Validation

6 criteria, all PASS. Overall: PASS WITH GAPS (gaps in open_questions/carry_forward, none blocking).

## Key observations

- Two-message-guards pattern (`_pending_messages` queue + `gateway/run.py` command interceptor) is the single highest-risk concurrency contract; any new always-reach command that misses either guard silently queues behind a running agent.
- Authorization has a one-flag trapdoor: `GATEWAY_ALLOW_ALL_USERS` and `TEAMS_ALLOW_ALL_USERS` flip default-deny to default-allow with a single env var.
- 58 slash commands resolves arch-CF1; downstream phases now have the full surface to reason about.

## Decisions Beyond Prompt

- **D1**: Read budget held at 8 files (vs 18-cap relaxation). 67KB commands.py parsed offline via grep+python regex to extract the 58 CommandDef rows.
- **D2**: Skills/, tests/, tools/*.py deep-dives skipped per brief; black-box scenarios sourced from AGENTS.md and architecture-map.md rather than direct test assertions.
- **D3**: Output ended at 33 KB vs 10-15 KB target - density driven by 34 contract rows + 12 scenarios + complete validation. No filler; kept because the relaxed brief said "complete-but-lean beats aspirational."

## Proposed Conventions

- **Conv-1**: "COMMAND_REGISTRY-style central dispatch tables are gold for contracts phases." When a target codebase has a single registry of commands/handlers, parse it into a table early - one regex extracts the entire surface and the contract rows largely write themselves.
- **Conv-2**: "Authorization trapdoor flags should always be flagged in the contracts phase, not deferred to defect-scan-security." Default-allow vs default-deny is a contract surface, not a defect.

## Citations

- Primary output: `.codecarto/findings/contracts/behavioral-contracts.md`
- Pipeline: `.codecarto/workflow/pipeline-full-with-deep-audit.yaml` (contracts phase)
- Open questions: ctr-OQ1 through ctr-OQ5
- Carry-forward: ctr-CF1 through ctr-CF11 (3 retroactive to protocols, 3 to defect-scan-semantic, 2 to porting, 2 to reimplementation-spec, 1 retroactive)
- Resolved arch-CF1 via 58-command catalog in public-surfaces append
