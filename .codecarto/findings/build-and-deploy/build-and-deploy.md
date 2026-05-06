# Build and Deploy

## 2026-05-06 — architecture phase

### Python packaging

- Build backend: `setuptools>=61.0` via `pyproject.toml`.
- Top-level `py-modules`: `run_agent`, `model_tools`, `toolsets`, `batch_runner`, `trajectory_compressor`, `toolset_distributions`, `cli`, `hermes_constants`, `hermes_state`, `hermes_time`, `hermes_logging`, `rl_cli`, `utils`.
- `setuptools.packages.find` includes: `agent`, `agent.*`, `tools`, `tools.*`, `hermes_cli`, `gateway`, `gateway.*`, `tui_gateway`, `tui_gateway.*`, `cron`, `acp_adapter`, `plugins`, `plugins.*`.
- Package data: `hermes_cli/web_dist/**/*` (built dashboard SPA).
- Python: 3.11+ required; Dockerfile uses 3.13.
- Lockfile: `uv.lock` (referenced by docker-publish workflow); `[tool.uv].exclude-newer = "7 days"` keeps uv reproducibility window short.
- Console scripts: `hermes`, `hermes-agent`, `hermes-acp`.
- Type-checker: `ty` configured under `[tool.ty]` (Python 3.13 target; broad rule overrides).
- Linter: `ruff` configured but `exclude = ["*"]` — effectively disabled at project level.
- Test runner: `pytest`, marker `integration`, `addopts = "-m 'not integration' -n auto"`.

### Optional extras (matrix)

`modal`, `daytona`, `vercel`, `dev`, `messaging`, `cron` (back-compat empty), `slack`, `matrix`, `cli`, `tts-premium`, `voice`, `pty` (PTY support, OS-conditional `ptyprocess` vs `pywinpty`), `honcho`, `mcp`, `homeassistant`, `sms`, `acp`, `mistral`, `bedrock`, `termux`, `dingtalk`, `feishu`, `google`, `web` (FastAPI+uvicorn), `rl` (atroposlib + tinker from git + wandb), `yc-bench` (git, py>=3.12), `all` (umbrella; excludes `[matrix]` on non-Linux).

### Node packaging

- Top-level `package.json` — browser tools (Playwright Chromium installed via `npx playwright install --with-deps chromium --only-shell`).
- `web/package.json` — dashboard SPA (Vite/React; `npm run build` outputs to `web/dist`, packaged into `hermes_cli/web_dist/`).
- `ui-tui/package.json` — Ink TUI host.
- `ui-tui/packages/hermes-ink/package.json` — inner package; rebuilt and copied to `node_modules/@hermes/ink` with React stripped (`rm -rf node_modules/@hermes/ink/node_modules/react`).
- Per-subdir `npm install --prefer-offline --no-audit`.

### Container image (`Dockerfile`)

- Multi-stage:
  - `uv_source` from `ghcr.io/astral-sh/uv:0.11.6-python3.13-trixie@sha256:b3c543...`.
  - `gosu_source` from `tianon/gosu:1.19-trixie@sha256:3b1766...`.
  - Final base `debian:13.4`.
- System deps: `build-essential curl nodejs npm python3 ripgrep ffmpeg gcc python3-dev libffi-dev procps git openssh-client docker-cli tini`.
- `tini` (PID-1) reaps zombies (MCP stdio subprocesses, git, bun, etc.).
- Non-root `hermes` user UID 10000 with home `/opt/data`. Runtime UID/GID remap via `gosu` in `docker/entrypoint.sh` (`HERMES_UID` / `HERMES_GID` env vars).
- `PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/.playwright` (outside the volume so survives mount).
- `HERMES_HOME=/opt/data` mounted as VOLUME.
- ENTRYPOINT: `/usr/bin/tini -g -- /opt/hermes/docker/entrypoint.sh`.
- Build steps: layer-cached npm installs → source COPY → npm builds (web + ui-tui) → `uv venv && uv pip install --no-cache-dir -e ".[all]"`.

### Compose (`docker-compose.yml`)

- Two services: `gateway` (`command: ["gateway", "run"]`) and `dashboard` (`command: ["dashboard", "--host", "127.0.0.1", "--no-open"]`).
- Both `network_mode: host`; volume mount `~/.hermes:/opt/data`.
- Optional API server requires uncommenting `API_SERVER_HOST` and `API_SERVER_KEY` (latter mandatory).
- Optional Teams variables commented out.
- Dashboard binds 127.0.0.1 by default; warning against LAN exposure without auth.

### Nix

- `flake.nix` at repo root.
- `.github/workflows/nix.yml` builds and tests the Nix derivation.
- `.github/workflows/nix-lockfile-fix.yml` (~10KB) manages flake lockfile.

### Install / setup scripts

- `scripts/install.sh` — `curl -fsSL ... | bash`. Linux/macOS/WSL2/Termux supported. Native Windows NOT supported.
- `setup-hermes.sh` — contributor clone-and-go: installs `uv`, creates venv, installs `.[all]`, symlinks `~/.local/bin/hermes`.
- `scripts/run_tests.sh` — hermetic test wrapper enforcing CI parity (unsets all `*_API_KEY`/`*_TOKEN`, TZ=UTC, LANG=C.UTF-8, `-n 4` xdist).
- `scripts/release.py` — release tooling (not read).

### CI/CD workflows (`.github/workflows/`)

| Workflow | Trigger | Purpose |
|---|---|---|
| `tests.yml` | push/PR to main (paths-ignore docs/md) | `test` job (uv venv + `.[all,dev]` + pytest -n auto, integration+e2e ignored, API keys unset); `e2e` job (separate) |
| `docker-publish.yml` | push to main / release / paths in py/Dockerfile/etc. | amd64 smoke test → multi-arch (amd64+arm64) push to Docker Hub `nousresearch/hermes-agent:latest` (main) or `:<tag>` (release). Gated to `github.repository == 'NousResearch/hermes-agent'`. |
| `nix.yml` | (paths/triggers not read) | Nix build/test |
| `nix-lockfile-fix.yml` | (~10KB) | Flake lockfile management |
| `deploy-site.yml` | Docs publishing | Docusaurus site to `hermes-agent.nousresearch.com/docs` |
| `docs-site-checks.yml` | PRs touching docs | Docs lint/build checks |
| `skills-index.yml` | Skill changes | Regenerate skills index |
| `supply-chain-audit.yml` | Schedule / dependency changes | Pinned-version audit (CVE checks per pyproject comments) |
| `contributor-check.yml` | PR open | Contributor agreement check |

### Distribution targets

| Target | Output |
|---|---|
| Docker Hub | `nousresearch/hermes-agent:latest` and `:<release-tag>` (multi-arch amd64+arm64) |
| Docusaurus site | `hermes-agent.nousresearch.com/docs` |
| GitHub Releases | (release-driven; `release.types: published` triggers Docker push) |
| `pip install -e .` (and extras) | Local dev / contributor path |
| `curl ... install.sh \| bash` | Quick install for end users |
| Termux curated path | `[termux]` extra (drops voice/matrix incompatibles) |
| Homebrew | (referenced indirectly via comments — `voice` extra excluded from base to keep source-build packagers happy) |
| Nix | Via `flake.nix` |

### Known build hazards

- `mautrix[encryption]` requires `python-olm` which is upstream-broken on modern macOS — hence `[matrix]` is gated to `sys_platform == 'linux'` in `[all]`.
- Voice extra pulls wheel-only deps (`ctranslate2`, `onnxruntime`) — incompatible with Termux.
- Native Windows unsupported; WSL2 only.
- Multi-arch Docker build cannot `load: true` for smoke testing — hence amd64-only smoke test before multi-arch push.
- Bind-mount UID mismatch in Docker — entrypoint usermod/groupmods + `gosu` to reconcile `HERMES_UID`/`HERMES_GID` with host owner.
