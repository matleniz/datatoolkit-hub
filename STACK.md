# Stack — allow-list of known tools

Rule: datatoolkit only uses tools Matteo knows. **Adding a dependency not
listed here needs Matteo's explicit approval first**, then a line here.

| Tool | Role | Where |
|---|---|---|
| uv | env + lockfile, one project per repo | `pyproject.toml` of each repo |
| pydantic v2 | key params + Result, JSON Schema export | engine |
| pandas | tabular data in keys | engine |
| plotly | figures (JSON contract) | engine, fronts |
| ruff | lint + import barrier (TID251) | gate |
| pytest | tests | gate |
| hatchling | build backend of both repos (invisible) | `pyproject.toml` |
| numpy | numeric arrays (ops, transforms, sklearn interop) | engine (declared direct 2026-09-28, MAT-187) |
| scikit-learn | fitted transforms (imputers, encoders, scalers), IsolationForest, `DtkTransformer` / `Pipeline` | engine (approved 2026-09-25) |
| pyarrow | parquet read / write (source + export) | engine (approved 2026-09-25) |
| openpyxl | Excel source | engine (approved 2026-09-25) |
| sqlalchemy | SQL source (URL from an env var) | engine (approved 2026-09-25) |
| FastAPI + uvicorn | HTTP API over the contract (`dtk-api`), optional extra `api` | engine (approved 2026-09-27, MAT-126) |
| httpx | FastAPI `TestClient` (tests only; transitive requirement) | engine dev |
| React + TypeScript + Vite | web front "Studio" (npm, one lockfile) | repo `datatoolkit-web` (approved 2026-09-27) |
| ESLint | lint of the web front | `datatoolkit-web` gate |
| Vitest | unit tests of the web front | `datatoolkit-web` gate (approved 2026-09-27) |
| Playwright | e2e tests with screenshots | `datatoolkit-web` (approved 2026-09-27) |
| react-grid-layout | Studio dock windows: drag to move + resize handles, snap-to-grid, serialisable layout | `datatoolkit-web` (approved 2026-09-29, MAT-234) |
| rapidfuzz | approximate string matching (spelling variants / aliases, approximate duplicates); ~3 MB wheel, no dependency | engine (approved 2026-09-28) |
| Docker + Docker Compose | engine + Studio images, one-command install (`compose.yml`) | both repos (approved 2026-09-30, MAT-255/256) |
| Base images python:3.12-slim, node:22-alpine, nginx:alpine | engine runtime; web build stage; web static server + `/api` proxy | Dockerfiles (approved 2026-09-30) |
| GitHub Actions + GHCR | PR smoke tests of the images, multi-arch publish of `ghcr.io/matleniz/datatoolkit-{engine,web}` | `.github/workflows/docker.yml` of both repos (approved 2026-09-30) |

Retired: streamlit (first front, repo `datatoolkit-streamlit` archived 2026-10-01).

Planned, not yet approved: polars, duckdb
(fast readers).
