# Course map — course section → capability → Linear issue

Source: Matteo's data course (Session 1 data collection, Session 2 data
preprocessing), raw pages kept locally in `documents/` (gitignored). Planned
2026-09-25. Status of each item: `CAPABILITIES.md`.

| Course page | What it teaches | datatoolkit capability | Issue |
|---|---|---|---|
| 1 Data and Its Shapes | grain ("one row = ?"), volume → tooling, provenance, raw / interim / processed | workspace export with provenance manifest; parquet column selection | MAT-47, MAT-40 |
| 2 Files and Formats | look at raw bytes first; csv `dtype` / `na_values` / `parse_dates` / `sep` / `decimal` / `header` / encoding; Excel sheets; parquet columns + partitions; json / jsonl + `json_normalize` | `file_inspect` key; csv options; `parquet`, `excel`, `json` readers | MAT-40 |
| 3 Databases | push work to the server, credentials in env vars, joins duplicate rows (assert row counts) | `sql` source (`url_env`, query pushed down); label join already refuses row loss | MAT-40 |
| 4 Preprocessing Contract | fit on train, transform everywhere; leaks; Pipeline; ColumnTransformer; CV inside the pipeline | fit/apply protocol, `DtkTransformer`, `workspace_pipeline` | MAT-39, MAT-47 |
| 5 Duplicates and Inconsistencies | exact / partial / approximate duplicates, conflicts, sort + keep last; casing, units, date formats, spellings, mixed types; `errors="raise"` | `duplicates`, `inconsistencies` keys; `drop_duplicates`, `standardize_text`, `parse_dates` ops | MAT-41, MAT-43 |
| 6 Missing Values | per-column / per-row missingness, sentinels, MCAR / MAR / MNAR, drop target-less rows, median / constant, ffill never bfill, KNN, iterative, indicator | `missing_values` key; `replace_sentinels`, `drop_missing_target`, `impute*`, `ffill` ops | MAT-42, MAT-43, MAT-44 |
| 7 Outliers | point / contextual / collective; z vs IQR; IsolationForest (`contamination` is an assumption); remove / clip / keep / transform | `outliers` key; `clip` op (fitted on train) | MAT-42, MAT-43 |
| 8 Categorical Encoding | nominal vs ordinal; one-hot (`handle_unknown`, `min_frequency`, cost); ordinal with explicit order | `onehot`, `ordinal` ops; advisor cardinality / cost | MAT-44, MAT-47 |
| 9 Feature Engineering | ratios (clip denominator), interactions, datetime parts, cyclical sin+cos, binning, group aggregations and their leak | `derive`, `interactions`, `datetime_parts`, `cyclical`, `bin`, `group_agg` ops | MAT-45 |
| 10 Scaling and Normalization | who needs scaling; standard / minmax / robust; skew → log1p; target scaling | `scale`, `log1p` ops; advisor per model family | MAT-44, MAT-47 |
| 11 Feature Selection | curse of dimensionality; filter / wrapper / embedded families (SelectKBest + mutual info, RFECV, Lasso); PCA fitted on scaled train, explained variance | `feature_selection` key; `drop_low_variance`, `drop_correlated`, `select_k_best`, `select_from_model`, `pca` ops | MAT-56 |

Not planned (ask first): approximate duplicate rows (rapidfuzz is used only for spelling variants in `inconsistencies`, `ops/consistency.py`),
NoSQL / MongoDB source, APIs / scraping, images / text manifests, target
transform (`TransformedTargetRegressor` belongs to modelling, out of scope).
