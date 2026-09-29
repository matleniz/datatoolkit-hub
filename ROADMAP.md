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
- 2026-09-25 — batches 1 + 2 merged (6 parallel workers, PRs datatoolkit#10–15):
  sources parquet / excel / json / sql + csv options + `file_inspect` (MAT-40),
  `duplicates` + `inconsistencies` (MAT-41), `missing_values` + `outliers`
  (MAT-42), 26 transform ops — cleaning (MAT-43), impute / encode / scale
  (MAT-44, KNN / iterative state = train matrix), features (MAT-45). Conflicts
  only on `keys/__init__.py` / `api.py` (unions). Gate: 292 tests. Catalog:
  `TRANSFORMS.md`. B3 dispatched (MAT-46 front, MAT-47 advisor / export).
- 2026-09-25 — front Transforms panel merged (datatoolkit-streamlit#1, MAT-46,
  26 front tests). Fleet bug: an inherited `FLEET_CONF` overrides `--project`
  (MAT-53); front dispatch uses `env -u FLEET_CONF -u FLEET_PROJECT`.
- 2026-09-25 — advisor + export merged (PR datatoolkit#16, MAT-47): key
  `preprocessing_advisor` (per-column plan as workspace steps, per model
  family), `export_workspace` (parquet + provenance manifest, big states in
  side files). Review fix before merge: overwrite only deletes validated paths
  of a `dtk_engine` manifest. Gate: 329 tests. **The course backlog
  (MAT-38 … MAT-47) is complete.**
- 2026-09-25 — follow-ups merged: `align_to_train` complete (PR #17, MAT-58:
  shift_mean / shift_median / standardize / robust / quantile, per-group,
  small-frame guard), `preview_workspace` in memory (PR #18 + front #2,
  MAT-55), feature selection from course 11 (PR #19, MAT-56: key +
  5 selection ops incl. PCA, `needs_target` for supervised ops in sklearn).
  Gate: engine 381 tests, front 27.

- 2026-09-26 — whole Linear backlog processed (coordinator + fleet workers, no
  human in the loop). Engine PRs datatoolkit#20–29: json/excel fixes (E2),
  robust csv + CSV-aware `file_inspect` with `load_spec` (E1), nested / binary
  columns + mixed date formats + typed feature_selection errors (E3), step
  validation at save / error taxonomy / single replay loop (E4), comparison
  keys `column_distribution`, `target_analysis`, `correlations` + column-selector
  convention + `source_columns` (C1, MAT-94), ops refactor into `advisor/` /
  `compare/` packages (E5), contract `needs_target` flags + `Table.kind` steps
  + engine doc honesty (E6), dogfood fixes (MAT-114 encoding on big samples,
  MAT-115 excel header), transform column-selector hints (MAT-119). Front PRs
  datatoolkit-streamlit#3–6: export / apply steps / needs_target / clean errors,
  column selectors + dataset input for any format with upload, workspace target
  prefill, Transforms-panel selectors. Gate: engine 523 tests, front 71.
  End-to-end check on a personal dataset (cp1252 `;` / decimal-comma csv +
  xlsx test, y in X): workspace → 12 keys → advisor steps applied → re-analyse
  → add / undo step → export, no error.

## 2026-09-27 — web front "Studio" (MAT-126)
Streamlit felt like a form catalog. Matteo validated a from-scratch design,
prototyped on Claude Design (Sources → train/test alignment → workbench:
pipeline bar, Excel-like grid, right-click, manual step editor showing what is
learned on train, variables + formulas, suggestions as a tab, dockable tools).
Approved: React + TS + Vite, FastAPI + uvicorn, Vitest, Playwright; new repo
`datatoolkit-web`; engine gains grid contract (MAT-127), `formula` op
(MAT-128), merges + variables (MAT-129), HTTP API (MAT-130). Spec: `FRONT-WEB.md`.

## 2026-09-28 — Studio built and e2e-validated
Web repo merged #1–#10 (scaffold, sources + alignment, workbench, panels +
dock, e2e suite), three screenshot-review rounds (MAT-139, MAT-142) and a
real-data dogfood (MAT-144: duplicate API calls cut, workbench open 21 s →
1.5 s on Parkinson). Engine #34: no sentinel alerts on identifier columns,
concise validation errors with `details`. Fleet gap found overnight (a worker
blocked 8 h on a foreground dev server) → MAT-140 (Agent Fleet).

## Next (see Linear)
- Studio UX review 2026-09-29 → umbrella MAT-230 (sub-issues MAT-231..241):
  left panel = Suggestions only (Variables / Recipe tabs removed),
  collapsible side panels, Transform in the tool rail, free resize / move of
  dock windows (with MAT-206), more freedom in every analysis window,
  visual-first Correlation / Outliers / Target / Missing / Chart windows,
  Python-style formulas. MAT-234 (layout lib), MAT-235 and MAT-241 need
  Matteo's validation before building.
- Waiting for Matteo's approval: MAT-74 (`.xls` needs `xlrd`), MAT-5 (FastAPI).
- Fleet bug MAT-53 (Agent Fleet) still forces `env -u FLEET_CONF -u FLEET_PROJECT`
  for front dispatches.
