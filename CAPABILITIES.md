# Capabilities — what datatoolkit will do, and status

Built key by key, on demand: every line below is validated with Matteo before
it is implemented. Architecture of the layers: `ARCHITECTURE.md`.

## Families

| Family | Shape | Examples | Status |
|---|---|---|---|
| **Analysis / check** | table(s) → `Result`, no table produced | overview, train/test consistency, distribution, duplicates, correlations | slice 1 |
| **Transform** | table(s) + params → table | join X_train + y_train (index / id), concat, derived feature (column combination), cast, drop, filter | later |
| **Pipeline** | JSON list of steps over a catalog of named tables | replay a preprocessing exactly (hash inputs + manifest) | later |

## Backlog

Status: `planned` (agreed direction) · `in progress` · `done` (card in `KEYS/`).

| Item | Family | What it answers | Status |
|---|---|---|---|
| source `csv` (pandas) | input | read any common CSV (sep sniffing, encoding, decimal) | done |
| `dataset_overview` | analysis | shape, memory, per-column dtype / semantic type / missing / uniques, head | done |
| harden `semantic_type` + fast sep sniffing | analysis / input | fewer false `id_like` (few non-null, all-distinct ints), "mostly numeric" text, years-as-datetime; sniff on first lines then C engine | planned (before / with slice 2) |
| Windows paths in `csv` source | input | accept `C:\…` and map it to `/mnt/c/…` (WSL) | planned |
| group-id detection in `semantic_type` | analysis | spot a repeated entity key (e.g. `patient_id`: 6 971 distinct / 55 603 rows) as `group_id`, not `categorical` | planned |
| `train_test_check` | analysis | columns on one side only (target), dtype mismatches, unseen categories, range and missing-rate shifts | planned (slice 2) |
| `column_distribution` | analysis | histogram / value counts of one column, train vs test overlay | planned |
| `duplicates` | analysis | duplicated rows, duplicated ids | planned |
| `correlations` | analysis | numeric correlation heatmap | planned |
| source `upload` | input | file uploaded in the front | planned |
| source `csv_robust` | input | malformed CSVs | planned |
| source `parquet` | input | polars / duckdb reader | planned, needs approval |
| `join`, `derive_feature`, … | transform | build new tables | design to discuss |
| pipeline runner | pipeline | reproducible replay | design to discuss |
