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
| test value spike (imputation) check | analysis | flag a value over-represented in test vs train (real data: `age_at_diagnosis` = 56.3 on 1 162 test rows) | idea, to discuss |
| `label_join_preview` | analysis | X and Y columns side by side, join candidates (uniqueness, match rate, row count, same order) before joining | planned |
| `column_distribution` | analysis | histogram / value counts of one column, train vs test overlay | planned |
| `duplicates` | analysis | duplicated rows, duplicated ids | planned |
| `correlations` | analysis | numeric correlation heatmap | planned |
| source `upload` | input | file uploaded in the front | planned |
| source `csv_robust` | input | malformed CSVs | planned |
| source `parquet` | input | polars / duckdb reader | planned, needs approval |
| transform ops (`drop_columns`, `rename`, `cast`, `fillna`, `filter_rows`, `derive`, realign test on train stats) | transform | modify train / test / both, logged in the workspace | design validated, ops to discuss one by one |
| pipeline runner | pipeline | reproducible replay of the workspace step log (+ input hashes, manifest) | later |
