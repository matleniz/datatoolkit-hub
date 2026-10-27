# How to run datatoolkit with Docker (engine + Studio, one command)

To share it, use the launcher (`HOWTO/share-and-run.md`): one line, Docker when
usable, else uv, and it opens the browser. Raw compose, by hand: any machine
with Docker (Docker Engine on Linux, Docker Desktop on macOS / Windows), no
clone, no Python or Node:

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

## Agent chat (opt-in)
Off by default. Set the env in the shell (or the `.env` next to the compose file):

    DTK_AGENT=1 DTK_UI_TOKEN=$(openssl rand -hex 16) ANTHROPIC_API_KEY=sk-... docker compose -f datatoolkit.yml up -d

`compose.yml` lists them as bare names: passed to the containers only when set,
so the default stack is unchanged.

| Env | Service | Role |
|---|---|---|
| `DTK_AGENT` | engine | `1` / `true` / `yes` / `on` turns the chat on (anything else: off) |
| `DTK_UI_TOKEN` | engine, web | Required when on (the engine exits 64 without it). The web container writes it into `index.html` as `<meta name="dtk-ui-token">` at start (`docker/40-dtk-ui-token.sh`), never in an image layer; unset = no meta, bridge off. URL-safe characters only (`A-Za-z0-9._~+/=-`) |
| `DTK_AGENT_PACK` | engine | `api-anthropic` (default), `api-openai`, `stub`; anything else exits 64 |
| `ANTHROPIC_API_KEY`, `DTK_ANTHROPIC_BASE_URL`, `DTK_ANTHROPIC_MODEL` | engine | `api-anthropic` |
| `DTK_OPENAI_BASE_URL`, `DTK_OPENAI_API_KEY`, `DTK_OPENAI_MODEL` | engine | `api-openai`; a host model is `http://host.docker.internal:11434/v1` (Linux: add `extra_hosts: ["host.docker.internal:host-gateway"]` to `engine` in an override) |
| `DTK_UI_ALLOWED_HOSTS`, `DTK_CORS_ORIGINS` | engine | Not needed behind the web nginx (same origin, `Host` forwarded); only for another hostname or TLS in front |

- Trust boundary: the token is in the served page, so the port stays
  `127.0.0.1` only. Do not publish it on another interface.
- nginx: `/api/ui/events` (SSE) unbuffered with a 1 h read timeout;
  `/api/ui/terminal`, `/mcp` and `/mcp/` return 404; the access log masks
  `token=`.
- The engine also keeps the token in `datatoolkit-data/agent/runtime.json`
  (0600, uid 1000); `DTK_UI_RUNTIME_FILE=0` on `engine` turns that off.
- No terminal / external CLI packs in Docker (`DTK_AGENT_TERMINAL` is always
  unset); the image has no agent CLI, no key, no `agent-sdk`.

## Limits
- A source given as an absolute **host** path is not visible in the
  container: use Studio's upload (it lands in `$DTK_HOME/uploads`).
- SQL sources read their URL from an env var: add it to the `engine`
  service's `environment` in your copy of `compose.yml`.

## CI (GitHub Actions, `.github/workflows/docker.yml` in both repos)
On every PR a `smoke` job (amd64) builds and runs the image(s): engine alone
(normal and root-owned bind mount); full compose for the web (index + SPA
fallback, `/api/keys` through nginx, 3 MB upload, `dataset_overview` run, file
owned by uid 1000 on the host); then agent off (no meta, no agent env, `/api/ui/agent/options` 401, `/mcp` and the terminal 404) and agent on with `DTK_AGENT=1 DTK_UI_TOKEN=ci-token DTK_AGENT_PACK=stub` (meta served, 401 without / wrong token, 200 with, a stub chat reply over SSE through nginx, token absent from the web image and from the logs). The engine smoke also covers the entrypoint contract (exit 64 without token, refused packs, no CLI, image size). On `main` the multi-arch `publish` job runs
after `smoke` and pushes `latest` + `sha-<short>` (engine also semver on `v*` tags).

No local Docker daemon on Matteo's WSL box (2026-09-30): verification lives in CI.
From a checkout, `docker compose up -d --build` builds both images locally
(engine from the git URL in `compose.yml`).
