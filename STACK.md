# Stack — allow-list of known tools

Rule: datatoolkit only uses tools Matteo knows. **Adding a dependency not
listed here needs Matteo's explicit approval first**, then a line here.

| Tool | Role | Where |
|---|---|---|
| uv | env + workspace (monorepo) | root `pyproject.toml` |
| pydantic v2 | key params + Result, JSON Schema export | engine |
| pandas | tabular data in keys | engine |
| plotly | figures (JSON contract) | engine, fronts |
| streamlit | first front | `front-streamlit` |
| ruff | lint + import barrier (TID251) | gate |
| pytest | tests | gate |
| hatchling | build backend of both repos (invisible) | `pyproject.toml` |
| scikit-learn | fitted transforms (imputers, encoders, scalers), IsolationForest, `DtkTransformer` / `Pipeline` | engine (approved 2026-09-25) |
| pyarrow | parquet read / write (source + export) | engine (approved 2026-09-25) |
| openpyxl | Excel source | engine (approved 2026-09-25) |
| sqlalchemy | SQL source (URL from an env var) | engine (approved 2026-09-25) |

Planned, not yet approved: FastAPI (HTTP API for a web front); polars, duckdb
(fast readers). Not approved: rapidfuzz (approximate duplicates) — ask first.
