# How to share datatoolkit (one command, with or without Docker)

The way to give datatoolkit to someone: one line, any machine (macOS, Linux,
Windows; amd64 or arm64). It starts the engine + Studio locally and opens the
browser on a demo workspace (`churn`).

```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/matleniz/datatoolkit-web/main/scripts/datatoolkit.sh | sh
```

```powershell
# Windows (PowerShell)
irm https://raw.githubusercontent.com/matleniz/datatoolkit-web/main/scripts/datatoolkit.ps1 | iex
```

The launcher (`scripts/datatoolkit.sh` / `datatoolkit.ps1` in `datatoolkit-web`,
datatoolkit-issues #53 / #55) picks the mode itself:

| | Docker mode (default when Docker is usable) | uv mode (no Docker, or `--uv` / `-Uv`) |
|---|---|---|
| Needs | Docker Desktop (macOS / Windows) or Docker Engine + compose plugin | nothing: uv is installed with Astral's installer if absent (user dir, no sudo); uv fetches Python |
| Runs | `compose.yml` in `$DTK_INSTALL_DIR` (default `~/datatoolkit`), public GHCR images, detached | `uv tool run --from "git+https://github.com/matleniz/datatoolkit-web#subdirectory=launcher" dtk-studio`, foreground (Ctrl+C / closing the window stops it) |
| URL | `http://localhost:${DTK_PORT:-8080}` | `http://localhost:8080`, or the next free port (printed) |
| Data | `${DTK_DATA:-$DTK_INSTALL_DIR/datatoolkit-data}` | `$DTK_HOME` (engine data home, default `~/.datatoolkit`) |
| Commands | `start` (default), `stop`, `update`, `uninstall [--purge]` (`-Purge`) | `start`, `update --uv` (refreshes the launcher and the Studio build); stop = Ctrl+C |
| First start | image pull | ~1 min (Python + deps, then the 1.6 MB Studio build) |

Piped form with a command: `curl … | sh -s -- stop`; PowerShell:
`& ([scriptblock]::Create((irm …/datatoolkit.ps1))) stop`.
Env: `DTK_NO_OPEN=1` (no browser), `DTK_TIMEOUT` (Docker readiness wait, 300 s),
`DTK_PORT` / `DTK_DATA` (persisted in the install dir's `.env`),
`DTK_COMPOSE_SRC` / `DTK_LAUNCHER_SRC` (CI overrides). `DTK_INSTALL_DIR` was
`DTK_HOME` before #55: `DTK_HOME` is the engine's data home. Both modes bind
`127.0.0.1` only (the engine reads local files and SQL URLs). The Docker mode
refuses an install dir that is a git checkout: on Matteo's machine `~/datatoolkit`
is the engine repo, so use `DTK_INSTALL_DIR=~/datatoolkit-app` there.

## uv mode internals
- `launcher/` in `datatoolkit-web`: package `dtk-studio` (hatchling), only
  dependency `dtk-engine[api] @ git+https://github.com/matleniz/datatoolkit`
  (nothing on PyPI). One ASGI app: the engine's `create_app()` for `/api` +
  Starlette `StaticFiles(html=True)` with SPA fallback; unknown `/api/*` stays
  404. Studio still talks to the engine over HTTP only.
- Studio build: `.github/workflows/studio-release.yml` publishes
  `studio-dist.zip` + `.sha256` to the rolling release `studio-latest` on every
  push to `main`. `dtk-studio` downloads it once, checks the sha256, unpacks it
  under `$DTK_HOME/studio/<sha>/` (`studio/current` names the one in use);
  `--refresh` looks for a newer build; offline with a cache, the cache is used.
  `DTK_STUDIO_URL` overrides the asset base URL (`file://` works).
- `dtk-studio [--port 8080] [--no-open] [--refresh]`.

## CI (datatoolkit-web)
- `docker.yml` → `launcher` job (ubuntu, `latest` images): shellcheck,
  PSScriptAnalyzer `PSUseCompatibleSyntax` 5.1, sh start twice (incl. piped),
  Studio index + `/api/keys` through nginx, stop, uninstall keeps data, custom
  port + `--purge`, ps1 end to end under pwsh.
- `uv-launcher.yml` (ubuntu, macOS, Windows): Studio built in the job and
  served over `file://`; sh one-liner with uv hidden + fresh HOME + dead Docker
  (fallback notice + uv install); ps1 piped `-Uv` on Windows PowerShell 5.1;
  `uvx --from ./launcher` with uv-managed Python; offline with cache and port
  8080 held (→ 8081). `scripts/smoke-uv-launcher.py` asserts `/`, a deep SPA
  route, `/api/keys`, a 404 under `/api`, then kills the tree.
- Checked by hand 2026-10-02 on WSL (no Docker daemon): the published sh
  one-liner with `--uv` on a fresh data home installed engine 853ecad + web
  172eca3, downloaded the release, served `/`, a deep route, `/api/keys`
  (13 keys), `/api/nope` → 404; headless Chromium opened Studio on the
  bootstrapped `churn` workspace with no page error.

Docker internals (images, compose, nginx, data volume): `HOWTO/run-with-docker.md`.
