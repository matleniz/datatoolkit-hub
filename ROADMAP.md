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

- 2026-09-25 — three slices merged: drift in `train_test_check` (PR #5,
  MAT-30: SMD, KS, PSI, % outside train p1–p99, TVD, overlaid histograms;
  p1–p99 info threshold raised 1 → 3 % after review), richer
  `dataset_overview` with tabs per semantic type (PR #6, MAT-29; Result
  `group` field), workspace + `dataset` source + label join + front top bar
  (PR #7, MAT-28). Gate: 168 tests on the combined merge. Real data: no drift;
  `age_at_diagnosis` test missing values imputed with a single value (56.3 on
  1 162 rows); labeled train 55 603 × 13, no row lost.

- 2026-09-25 — course backlog planned from Matteo's data collection /
  preprocessing course (`COURSE-MAP.md`). Decisions: split into two repos
  (engine alone installable, Streamlit apart), three doors to one `ops/`
  implementation (JSON contract, notebook `api`, sklearn `DtkTransformer`),
  fit/apply transform protocol; scikit-learn, pyarrow, openpyxl, sqlalchemy
  approved. Linear MAT-38 … MAT-47 in 4 batches: B0 split + foundations
  (sequential, opus) → B1 sources + quality keys ∥ B2 transform ops → B3
  front panel + advisor/export.
- 2026-09-25 — repo split merged (PR datatoolkit#8, MAT-38): engine flattened
  at the root of `datatoolkit` (147 tests), front in new private repo
  `datatoolkit-streamlit` (21 tests, own fleet project, same hub and Linear
  queue), front pinned to engine `main`.
- 2026-09-25 — foundations merged (PR datatoolkit#9, MAT-39): fit/apply
  transform protocol + replay, `list_transforms` / `transform_schema`,
  notebook `api` + `Result._repr_html_`, `DtkTransformer` /
  `workspace_pipeline`, deps sklearn / pyarrow / openpyxl / sqlalchemy.
  Gate: 172 tests. Batch 1 + 2 dispatched (6 workers, MAT-40 … MAT-45).

## Next (see Linear)
- B1 ∥ B2 after MAT-39; B3 last. HTTP API (MAT-5) still needs FastAPI approval.
