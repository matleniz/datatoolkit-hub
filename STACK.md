# Stack — allow-list of known tools

Rule: datatoolkit only uses tools Matteo knows. **Adding a dependency not
listed here needs Matteo's explicit approval first**, then a line here.

| Tool | Role | Where |
|---|---|---|
| uv | env + lockfile, one project per repo | `pyproject.toml` of each repo |
| pydantic v2 | key params + Result, JSON Schema export | engine |
| pandas | tabular data in keys | engine |
| plotly | figures (JSON contract) | engine, fronts |
| streamlit | first front | repo `datatoolkit-streamlit` |
| ruff | lint + import barrier (TID251) | gate |
| pytest | tests | gate |
| hatchling | build backend of both repos (invisible) | `pyproject.toml` |
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

Planned, not yet approved: polars, duckdb
(fast readers). Not approved: rapidfuzz (approximate duplicates) — ask first.
