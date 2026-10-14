# Stack — allow-list of known tools

Rule: datatoolkit only uses tools Matteo knows. **Adding a dependency not
listed here needs Matteo's explicit approval first**, then a line here.

| Tool | Role | Where |
|---|---|---|
| uv | env + lockfile, one project per repo | `pyproject.toml` of each repo |
| pydantic v2 | key params + Result, JSON Schema export | engine |
| pandas | tabular data in keys | engine |
| plotly | figures (JSON contract) | engine, fronts |
| ruff | lint: import sorting, complexity ≤ 10 (`C90`, `PLR09xx`), `SIM`, `PERF`, `B` (layer contract = `tests/test_layers.py`) | engine gate |
| pytest | tests | gate; engine CI runs it on Python 3.11, 3.12 and 3.13 (datatoolkit-issues#208, #216) |
| pytest-cov | test coverage (`uv run pytest --cov=dtk_engine --cov-report=term-missing`; 96% on 2026-10-14), no gate | engine dev group only, never in the Docker image (approved 2026-10-14, datatoolkit-issues#209) |
| hatchling | build backend of the engine (invisible) | `pyproject.toml` |
| numpy | numeric arrays (ops, transforms, sklearn interop) | engine (declared direct 2026-09-28, MAT-187) |
| scikit-learn | fitted transforms (imputers, encoders, scalers), IsolationForest, `DtkTransformer` / `Pipeline` | engine (approved 2026-09-25) |
| pyarrow | parquet read / write (source + export) | engine (approved 2026-09-25) |
| openpyxl | Excel source | engine (approved 2026-09-25) |
| sqlalchemy | SQL source (URL from an env var) | engine (approved 2026-09-25) |
| FastAPI + uvicorn | HTTP API over the contract (`dtk-api`), optional extra `api` | engine (approved 2026-09-27, MAT-126) |
| httpx | FastAPI `TestClient` (dev) and HTTP client of the direct API chat packs | engine dev + extras `agent`, `agent-sdk` |
| rapidfuzz | approximate string matching (spelling variants / aliases, approximate duplicates); ~3 MB wheel, no dependency | engine (approved 2026-09-28) |
| mcp (official Python MCP SDK) | MCP server over the contract (streamable HTTP at `/mcp` + stdio `dtk-mcp`), optional extra `agent`; transitive deps listed in the PR that adds it | engine (approved 2026-10-02, datatoolkit-issues#64, `AGENT-BRIDGE.md`) |
| claude-agent-sdk (Python) | first in-Studio agent pack (agent loop, permissions, tool allow-list), optional extra `agent-sdk` | engine (approved 2026-10-02, datatoolkit-issues#67, `AGENT-BRIDGE.md`) |
| websockets + httpx (promoted from dev to the extras `agent` and `agent-sdk`) | PTY bridge for the terminal pack; direct API chat pack (Anthropic Messages or any OpenAI-compatible endpoint, one HTTP client, no vendor SDK) | engine, extras `agent-sdk` / `agent` (approved 2026-10-03, Matteo asked for an API or CLI choice; stdlib `pty` on Linux / macOS / WSL, no `pywinpty`) |
| Docker + Docker Compose | engine + Studio images, one-command install (`compose.yml`) | both repos (approved 2026-09-30, MAT-255/256) |
| Base images python:3.12-slim, node:22-alpine, nginx:alpine | engine runtime; web build stage; web static server + `/api` proxy | Dockerfiles (approved 2026-09-30) |
| GitHub Actions + GHCR | PR smoke tests of the images, multi-arch publish of `ghcr.io/matleniz/datatoolkit-{engine,web}` | `.github/workflows/docker.yml` of both repos (approved 2026-09-30) |

| pypdf | text extraction of PDF workspace documents (agent + Studio) | engine, optional extra (approved 2026-10-06, datatoolkit-issues#178) |

Web front (Studio) tools: allow-list in `~/datatoolkit-web-hub` `STACK.md`.

Retired: streamlit (first front, repo `datatoolkit-streamlit` archived 2026-10-01).

Planned, not yet approved: polars, duckdb
(fast readers).
