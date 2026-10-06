# Capabilities — what datatoolkit will do, and status

Built key by key, on demand: every line below is validated with Matteo before
it is implemented. Architecture of the layers: `ARCHITECTURE.md`.

## Families

| Family | Shape | Examples | Status |
|---|---|---|---|
| **Analysis / check** | table(s) → `Result`, no table produced | overview, train/test consistency, distribution, duplicates, correlations | built |
| **Transform** | table(s) + params → table | join X_train + y_train (index / id), concat, derived feature (column combination), cast, drop, filter | step catalog built (label join, cleaning, align, impute / encode / scale, features, selection, formula — `TRANSFORMS.md`) |
| **Pipeline** | JSON list of steps over a catalog of named tables | replay a preprocessing exactly (hash inputs + manifest) | later |

## Backlog

Status: `planned` (agreed direction) · `in progress` · `done` (card in `KEYS/`).

| Item | Family | What it answers | Status |
|---|---|---|---|
| source `csv` (pandas) | input | read any common CSV (sep sniffing, auto encoding / decimal, leading zeros kept, strict bad-line check) | done — MAT-61…69 |
| `dataset_overview` | analysis | shape, memory, per-column dtype / semantic type / missing / uniques, head | done |
| harden `semantic_type` + fast sep sniffing | analysis / input | fewer false `id_like` (few non-null, all-distinct ints), "mostly numeric" text, years-as-datetime; sniff on first lines then C engine | done |
| Windows paths in `csv` source | input | accept `C:\…` and map it to `/mnt/c/…` (WSL) | done |
| group-id detection in `semantic_type` | analysis | spot a repeated entity key (e.g. `patient_id`: 6 971 distinct / 55 603 rows) as `group_id`, not `categorical` | done |
| `train_test_check` | analysis | columns on one side only (target), dtype mismatches, unseen categories, range and missing-rate shifts, train/test overlap | done |
| workspace + source `dataset` + front top bar | input / state | keep X_train / y_train / X_test in memory across pages (JSON workspace), y separate or already in X, label join by order or key | done |
| richer `dataset_overview` | analysis | numeric stats (mean, std, quantiles, skew, kurtosis, zeros, negatives, IQR / z outliers), browsable category values, results in one tab per semantic type | done |
| train/test drift in `train_test_check` | analysis | per numeric column train vs test stats side by side, standardized mean diff, KS, PSI, % test outside train p1–p99, overlaid histograms; category frequencies | done |
| `label_join_preview` | analysis | X and Y columns side by side, join candidates (uniqueness, match rate, row count, same order) before joining | done — datatoolkit-issues#58 (engine #84) |
| `column_distribution` | analysis | histogram / value counts of picked columns, train vs test overlay, split by label or by any other column | done — MAT-95, MAT-159 |
| `target_analysis` | analysis | each feature vs the label: ranking, per-class stats, class rate per category, binned mean target | done — MAT-96 |
| `correlations` | analysis | pearson / spearman heatmap over picked numeric columns + pairs above a threshold (drop_correlated rule, suggested step) | done — MAT-97 |
| source `upload` | input | file uploaded in the front (saved under `$DTK_HOME/uploads`, then a normal file source) | done — MAT-102 |
| source `csv_robust` | input | malformed CSVs | done — bad lines strict / recover (MAT-68/69); junk header lines (`skiprows`) and mixed separators (`mixed_sep`) (datatoolkit-issues#59, engine #85). Not covered: junk lines with the body's exact field count, mixed separators on quoted lines |
| repo split: engine-only `datatoolkit` + `datatoolkit-streamlit` | architecture | engine installable alone (notebook, scripts) | done — MAT-38 (front repo archived 2026-10-01) |
| fit/apply transform protocol, `list_transforms` / `transform_schema`, notebook `api`, `DtkTransformer`, `workspace_pipeline` | architecture | same ops from front, notebook and sklearn `Pipeline`, no leak | done — MAT-39 |
| sources: csv `na_values` / `dtype` / `parse_dates`, `parquet`, `excel`, `json` / `jsonl`, `sql` (URL via env var) + `file_inspect` key | input | read every course format; look at raw bytes before loading | done — MAT-40 |
| `duplicates` + `inconsistencies` | analysis | exact / partial duplicates, key conflicts; casing / whitespace variants, mixed types, ambiguous dates, suggested mapping | done — MAT-41 |
| `missing_values` + `outliers` | analysis | missing rate, per-row missing spikes, sentinels, co-occurrence, test imputation spike; IQR / z / IsolationForest | done — MAT-42 |
| cleaning ops (`drop_columns`, `select_columns`, `rename`, `rename_columns_bulk`, `reorder_columns`, `copy_column`, `sort_rows`, `sample_rows`, `replace_values`, `cast`, `drop_duplicates`, `standardize_text`, `parse_dates`, `replace_sentinels`, `drop_missing_target`, `filter_rows`, `clip`, `align_to_train`) | transform | fix the defects found by the analyses; `align_to_train` options to confirm with Matteo | done — MAT-43 |
| imputation / encoding / scaling ops (`impute` + indicator, `impute_knn`, `impute_iterative`, `ffill`, `onehot`, `ordinal`, `scale`, `log1p`) | transform | model-ready matrix, fitted on train | done — MAT-44 |
| feature ops (`derive`, `datetime_parts`, `cyclical`, `bin`, `group_agg`, `interactions`) | transform | domain features without leak | done — MAT-45 |
| Streamlit Transforms panel | front | add / undo steps generically, before/after preview | done — MAT-46, MAT-55; Streamlit archived 2026-10-01, Studio has its own |
| front: export button, apply result steps, needs_target, clean errors | front | finish the loop analyse → apply → export from the UI | done — MAT-91/92/93 |
| front: column selectors, dataset input (any format, upload, options), workspace target prefill | front | analyse a personal dataset end to end without typing column names or JSON | done — MAT-98, MAT-102, MAT-118 |
| `preprocessing_advisor` + workspace export (parquet + manifest) | analysis / pipeline | per-column recommendation → op; reproducible, provenance-tracked output | done — MAT-47 |
| `align_to_train` complete (median / robust / quantile modes, per-group, small-frame guard) | transform | realign a shifted test column on train's distribution | done — MAT-58 |
| `feature_selection` key + selection ops (`drop_low_variance`, `drop_correlated`, `select_k_best`, `select_from_model`, `pca`) | analysis / transform | which columns to keep; reduce width without leak | done — MAT-56 |
| step preview without store write (`preview_workspace`) | contract / front | before / after of a pending step, no scratch workspace | done — MAT-55 |
| pipeline runner | pipeline | reproducible replay of the workspace step log (+ input hashes, manifest) | later |
