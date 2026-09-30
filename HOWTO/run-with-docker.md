# How to run datatoolkit with Docker (engine + Studio, one command)

Any machine with Docker (Docker Engine on Linux, Docker Desktop on macOS /
Windows), no clone, no Python or Node:

```bash
curl -fsSL https://raw.githubusercontent.com/matleniz/datatoolkit-web/main/compose.yml -o datatoolkit.yml && docker compose -f datatoolkit.yml up -d
```

PowerShell: same with `curl.exe`. Then open http://localhost:8080.
Update: `docker compose -f datatoolkit.yml pull && docker compose -f datatoolkit.yml up -d`.

## What runs
| Service | Image (GHCR, public, amd64 + arm64) | Repo files |
|---|---|---|
| `engine` | `ghcr.io/matleniz/datatoolkit-engine` — `dtk-api --host 0.0.0.0 --port 8765`, `DTK_HOME=/data`, healthcheck on `/api/keys` | engine `Dockerfile`, `docker/entrypoint.sh` |
| `web` | `ghcr.io/matleniz/datatoolkit-web` — nginx serving the Vite build, `/api/` proxied to `engine:8765` (same origin, no CORS) | web `Dockerfile`, `docker/nginx.conf`, `compose.yml` |

- Port: `127.0.0.1:${DTK_PORT:-8080}` only (the engine reads files and SQL URLs:
  not exposed to the LAN on purpose). The engine publishes no port.
- Data: `${DTK_DATA:-./datatoolkit-data}` bind-mounted on `/data`
  (workspaces, uploads, exports visible on the host). The engine entrypoint
  starts as root, chowns `/data` to uid 1000 when the daemon created it as
  root, then drops to uid 1000 (`setpriv`).
- nginx: `client_max_body_size 0` (uploads are raw `PUT /api/uploads/{name}`
  bodies), 300 s proxy timeouts, SPA fallback to `index.html`.

## Limits
- A source given as an absolute **host** path is not visible in the
  container: use Studio's upload (it lands in `$DTK_HOME/uploads`).
- SQL sources read their URL from an env var: add it to the `engine`
  service's `environment` in your copy of `compose.yml`.

## CI (GitHub Actions, `.github/workflows/docker.yml` in both repos)
On every PR a `smoke` job (amd64) builds and runs the image(s): engine alone
(normal and root-owned bind mount); full compose for the web (index + SPA
fallback, `/api/keys` through nginx, 3 MB upload, `dataset_overview` run, file
owned by uid 1000 on the host). On `main` the multi-arch `publish` job runs
after `smoke` and pushes `latest` + `sha-<short>` (engine also semver on `v*` tags).

No local Docker daemon on Matteo's WSL box (2026-09-30): verification lives in CI.
From a checkout, `docker compose up -d --build` builds both images locally
(engine from the git URL in `compose.yml`).
