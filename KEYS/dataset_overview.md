# Key `dataset_overview` — Dataset overview

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/dataset_overview.py` · test `tests/keys/test_dataset_overview.py`
- **What it answers:** what does this table look like: size, memory, missing values, duplicates, the type / cardinality of each column, then per-type statistics (numeric distribution and outliers, category values, dates, text, ids)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | `SourceSpec` (discriminated on `kind`: `csv`, `parquet`, `excel`, `json`, `sql`, `dataset` — `sources/spec.py`) | `{"kind": "csv", "path": <demo_data/train.csv>}` | table to load. `csv`: `path`, `sep` (`"auto"` sniffs), `encoding` (`utf-8`), `decimal` (`.`), `header` (`0`, null = no header) |
| `head_rows` | int, 1..1000 | 5 | rows returned in the `head` table |

## Result
One tab per group (Result `group` field); a group exists only when the table has columns of that semantic type.
- metrics: `rows`, `cols`, `memory_mb` (deep, 3 decimals), `pct_missing_cells` (0..100), `n_duplicate_rows` (exact duplicate rows)
- **Overview**: table `columns` — one record per column `{column, dtype, semantic_type, n_missing, pct_missing, n_unique, pct_numeric_parsable, sample_values}` (`ops.profile.column_profile`); table `head` — first `head_rows` rows; figure `% missing per column`
- **Numeric**: table `numeric stats` — count, mean, std, min, p1, p5, q1, median, q3, p95, p99, max, skew, kurtosis, n_zeros, n_negative, n_outliers_iqr / pct_outliers_iqr (outside q1 − 1.5 IQR / q3 + 1.5 IQR), n_outliers_z / pct_outliers_z (|z| > 3); pct over non-null; figure `distributions` — 30-bin histogram grid, one facet per column
- **Categorical** (categorical + boolean): `category summary` — n_unique, top_value, top_pct, pct_rare (% of rows in values covering < 1 % each); `category values` — long `{column, value, count, pct}`, missing shown as `(missing)`, sorted by column then count desc (filterable per column in the front)
- **Datetime**: `datetime stats` — count, min, max, span_days
- **Text**: `text stats` — count, n_unique, mean / min / max length
- **IDs** (id_like + group_id): `id stats` — count, n_unique, n_duplicates, pct_duplicates

## Notes
- `semantic_type` (`ops/profile.py`) is a heuristic in {numeric, categorical, boolean, datetime, text, id_like, group_id, constant}: ≤1 distinct non-null → constant; numeric {0,1} → boolean; **id_like** = ≥ 20 non-null values (`ID_MIN_NON_NULL`) and distinct/non-null ≥ 0.95, plus whitespace-free for strings or, for integers, distinct values filling ≥ 95 % of their [min, max] range (`Index` 0..n-1 yes, spread-out all-distinct ints → numeric); **group_id** (repeated entity key, e.g. `patient_id`) = ≥ 100 distinct values with ratio in [0.01, 0.95), whitespace-free strings or integers filling ≥ 50 % of their range; strings fully parseable as ISO-8601 → datetime, except pure-digit strings (years); strings with distinct ratio > 0.5 → text, else categorical. Low-cardinality integers (e.g. `Pclass`) stay numeric. Thresholds are module constants.
- `pct_numeric_parsable` = % of non-null values that read as numbers (100 for numeric dtypes, 0 for bool/datetime). A `text`/`categorical` column scoring high is numbers polluted by tokens (demo test `Age`: 85.0 because of "unknown").
- Remaining weak spots: a sparse all-distinct string column (demo `Cabin`, 5 values) reads `text`; an integer count feature with ≥ 100 dense distinct values on a large table can read `group_id`.
- Checked on real data (X_train 55 603 × 12): `Index` → id_like, `patient_id` → group_id, `sexM` → boolean.
- A missing or unreadable file raises `SourceError` (not `KeyParamsError`); an unknown `kind` or misspelled param raises `KeyParamsError`.
- ops functions: `numeric_stats`, `numeric_histograms`, `category_values`, `category_summary`, `datetime_stats`, `text_stats`, `id_stats`, `semantic_types`, `columns_of_type` (constants `RARE_PCT`, `OUTLIER_IQR_K`, `OUTLIER_Z`, `HIST_BINS`).
- Real data X_train: ~2 s; 7 numeric, 3 categorical, 2 id columns (`patient_id` 87 % repeated rows).
- Default demo train: 41 rows × 12 cols, 1 deliberate duplicate row.
