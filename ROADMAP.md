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

- Hardening merged (PR datatoolkit#3, MAT-24): `group_id` type, fewer false
  `id_like`, `pct_numeric_parsable`, years not datetime, Windows paths, fast
  sniffing (X_train read 1.0 s → 0.46 s). Gate: 69 tests.

- Slice 2 merged (PR datatoolkit#4, MAT-25): `train_test_check` (`ops/compare.py`:
  schema, missing, ranges, categories, row / id overlap, severity-ranked
  issues); `hello` removed. Review fix before merge: row overlap now ignores
  ids / row counters, int vs float dtype = info. Gate: 91 tests. On the real
  X_train / X_test: `time_since_diagnosis` only in test, `age_at_diagnosis`
  5 % → 0 % missing, 23 identical feature rows across splits (different
  patients, sparse rows), no `patient_id` leak.

- 2026-09-25 — workspace design validated (engine-side JSON, step log =
  future pipeline, `kind="dataset"` source, tabs per semantic type, stat ops
  can fit on train). Three parallel workers: workspace, richer overview stats,
  train/test drift.

## Next (see Linear)
- HTTP API (FastAPI, 3 generic routes) + `HttpClient` — needs FastAPI approval.
- `label_join_preview`, then transform ops one by one.
