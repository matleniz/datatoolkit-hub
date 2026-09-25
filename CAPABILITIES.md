# Capabilities — what datatoolkit will do, and status

Built key by key, on demand: every line below is validated with Matteo before
it is implemented. Architecture of the layers: `ARCHITECTURE.md`.

## Families

| Family | Shape | Examples | Status |
|---|---|---|---|
| **Analysis / check** | table(s) → `Result`, no table produced | overview, train/test consistency, distribution, duplicates, correlations | built |
| **Transform** | table(s) + params → table | join X_train + y_train (index / id), concat, derived feature (column combination), cast, drop, filter | label join built (workspace); step ops next |
| **Pipeline** | JSON list of steps over a catalog of named tables | replay a preprocessing exactly (hash inputs + manifest) | later |

## Backlog

Status: `planned` (agreed direction) · `in progress` · `done` (card in `KEYS/`).

| Item | Family | What it answers | Status |
|---|---|---|---|
| source `csv` (pandas) | input | read any common CSV (sep sniffing, encoding, decimal) | done |
| `dataset_overview` | analysis | shape, memory, per-column dtype / semantic type / missing / uniques, head | done |
| harden `semantic_type` + fast sep sniffing | analysis / input | fewer false `id_like` (few non-null, all-distinct ints), "mostly numeric" text, years-as-datetime; sniff on first lines then C engine | done |
| Windows paths in `csv` source | input | accept `C:\…` and map it to `/mnt/c/…` (WSL) | done |
| group-id detection in `semantic_type` | analysis | spot a repeated entity key (e.g. `patient_id`: 6 971 distinct / 55 603 rows) as `group_id`, not `categorical` | done |
| `train_test_check` | analysis | columns on one side only (target), dtype mismatches, unseen categories, range and missing-rate shifts, train/test overlap | done |
| workspace + source `dataset` + front top bar | input / state | keep X_train / y_train / X_test in memory across pages (JSON workspace), y separate or already in X, label join by order or key | done |
| richer `dataset_overview` | analysis | numeric stats (mean, std, quantiles, skew, kurtosis, zeros, negatives, IQR / z outliers), browsable category values, results in one tab per semantic type | done |
| train/test drift in `train_test_check` | analysis | per numeric column train vs test stats side by side, standardized mean diff, KS, PSI, % test outside train p1–p99, overlaid histograms; category frequencies | done |
| `label_join_preview` | analysis | X and Y columns side by side, join candidates (uniqueness, match rate, row count, same order) before joining | planned |
| `column_distribution` | analysis | histogram / value counts of one column, train vs test overlay | planned |
| `correlations` | analysis | numeric correlation heatmap | planned |
| source `upload` | input | file uploaded in the front | planned |
| source `csv_robust` | input | malformed CSVs | planned |
| repo split: engine-only `datatoolkit` + `datatoolkit-streamlit` | architecture | engine installable alone (notebook, scripts) | done — MAT-38 |
| fit/apply transform protocol, `list_transforms` / `transform_schema`, notebook `api`, `DtkTransformer`, `workspace_pipeline` | architecture | same ops from front, notebook and sklearn `Pipeline`, no leak | planned — MAT-39 |
| sources: csv `na_values` / `dtype` / `parse_dates`, `parquet`, `excel`, `json` / `jsonl`, `sql` (URL via env var) + `file_inspect` key | input | read every course format; look at raw bytes before loading | planned — MAT-40 |
| `duplicates` + `inconsistencies` | analysis | exact / partial duplicates, key conflicts; casing / whitespace variants, mixed types, ambiguous dates, suggested mapping | planned — MAT-41 |
| `missing_values` + `outliers` | analysis | missing rate, per-row missing spikes, sentinels, co-occurrence, test imputation spike; IQR / z / IsolationForest | planned — MAT-42 |
| cleaning ops (`drop_columns`, `rename`, `cast`, `drop_duplicates`, `standardize_text`, `parse_dates`, `replace_sentinels`, `drop_missing_target`, `filter_rows`, `clip`, `align_to_train`) | transform | fix the defects found by the analyses; `align_to_train` options to confirm with Matteo | planned — MAT-43 |
| imputation / encoding / scaling ops (`impute` + indicator, `impute_knn`, `impute_iterative`, `ffill`, `onehot`, `ordinal`, `scale`, `log1p`) | transform | model-ready matrix, fitted on train | planned — MAT-44 |
| feature ops (`derive`, `datetime_parts`, `cyclical`, `bin`, `group_agg`, `interactions`) | transform | domain features without leak | planned — MAT-45 |
| Streamlit Transforms panel | front | add / undo steps generically, before/after preview | planned — MAT-46 |
| `preprocessing_advisor` + workspace export (parquet + manifest) | analysis / pipeline | per-column recommendation → op; reproducible, provenance-tracked output | planned — MAT-47 |
| pipeline runner | pipeline | reproducible replay of the workspace step log (+ input hashes, manifest) | later |
