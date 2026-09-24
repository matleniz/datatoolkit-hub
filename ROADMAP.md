# Roadmap — dated journal (historical)

## 2026-09-24 — bootstrap
- Linear project `datatoolkit` (team MAT), fleet project `datatoolkit`
  (queue Linear, gate = ruff + pytest).
- Hub seeded: architecture/contract, stack allow-list, key cards, howtos.
- Skeleton built by a Sonnet worker (`dtk-skeleton`, PR #1, gate green, reviewed): engine + contract +
  `hello` key + generic Streamlit front + isolation barrier.

## 2026-09-24 — design: analysis / transform / pipeline
- Scope agreed with Matteo: analysis first, then transforms and reproducible
  pipelines; plug-and-play inputs (clean CSV with pandas now; malformed CSV,
  parquet via polars/duckdb, upload later). See `CAPABILITIES.md`.
- Engine split into sources → ops → keys; `SourceSpec` discriminated on `kind`;
  strict params; recursive generic front forms (`ARCHITECTURE.md`).
- Slice 1 merged (PR datatoolkit#2, MAT-8): csv source, `ops/profile`,
  `dataset_overview`, demo train/test CSVs, strict `KeyParams`, recursive
  front forms (`schema.py`). Gate: ruff + 42 tests. Review follow-ups:
  `semantic_type` heuristics, slow `sep="auto"` on big files.

## Next (see Linear)
- HTTP API (FastAPI, 3 generic routes) + `HttpClient` — needs FastAPI approval.
- Slice 2: `train_test_check` (after slice 1 review).
