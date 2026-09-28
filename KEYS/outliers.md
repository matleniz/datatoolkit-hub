# Key `outliers` — Outliers

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/outliers.py` · tests `tests/keys/test_outliers.py`, `tests/ops/test_outlier_ops.py`
- **Notebook:** `api.outliers(…)`
- **What it answers:** Which numeric values fall outside IQR fences or |z| thresholds, and which rows are anomalous as a combination (IsolationForest)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | table to check |
| `method` | `all` \| `iqr` \| `zscore` \| `isolation_forest` | `all` | restrict which detector(s) run (MAT-174) |
| `iqr_k` | float | 1.5 | IQR fence multiplier (Tukey) |
| `z_threshold` | float | 3.0 | flag values with \|z\| above this |
| `contamination` | float | 0.01 | expected share of anomalous rows — an assumption you make, no `auto` |
| `random_state` | int | 0 | IsolationForest seed |

## Result
- metrics: `n_rows`, `n_numeric_columns`, `n_columns_with_iqr_outliers`, `n_columns_with_z_outliers`, `n_rows_flagged`, `contamination`
- tables: `outliers_per_column` (fences, n / % IQR, n / % z), `flagged_rows` (20 most anomalous)
- figures: % outside fences per column, anomaly score histogram
- text: the course's remove / clip / keep / transform decision table

## Notes
Ops: `ops/outliers.py`. Screens `numeric` semantic columns only (ids and constants excluded); IsolationForest median-fills gaps. Fix with `clip` (bounds fitted on train), `log1p`, or `scale(method=robust)`.
