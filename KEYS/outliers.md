# Key `outliers` — Outliers

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/outliers.py` · tests `tests/keys/test_outliers.py`, `tests/ops/test_outlier_ops.py`
- **Notebook:** `api.outliers(…)` (`mad_threshold` is the last argument)
- **What it answers:** Which numeric values fall outside IQR fences, |z| or robust MAD z-score thresholds, and which rows are anomalous as a combination (IsolationForest)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | table to check |
| `columns` | list[str] (column selector, numeric) | `[]` | empty = every numeric column; otherwise restrict IQR / z / IsolationForest to this pick |
| `method` | `all` \| `iqr` \| `zscore` \| `mad` \| `isolation_forest` | `all` | restrict which detector(s) run (MAT-174); `mad` is opt-in and not part of `all`, so existing results stay unchanged (#291) |
| `iqr_k` | float | 1.5 | IQR fence multiplier (Tukey) |
| `z_threshold` | float | 3.0 | flag values with \|z\| above this |
| `mad_threshold` | float (> 0) | 3.5 | flag values with robust z `0.6745*(x-median)/MAD` above this in absolute value (`method=mad`); MAD = 0 flags nothing; NaN ignored; masked-by-outliers problem of the mean z-score avoided (#291) |
| `contamination` | float | 0.01 | expected share of anomalous rows — an assumption you make, no `auto` |
| `random_state` | int | 0 | IsolationForest seed |

## Result
- metrics: `n_rows`, `n_numeric_columns`, `n_columns_with_iqr_outliers`, `n_columns_with_z_outliers`, `n_columns_with_mad_outliers`, `n_rows_flagged`, `contamination`
- tables: `outliers_per_column` (fences, n / % IQR, n / % z, n_mad / pct_mad), `flagged_rows` (20 most anomalous), `box_stats` (per column: q1, median, q3, lower / upper fence and whisker, n_below, n_above; when the method includes IQR, MAT-237)
- figures: **main** = one column selected → box plot of that column with both IQR fences drawn and outlier points highlighted (always the box, even with 0 outliers); several columns → horizontal bars of % outside fences, sorted, only columns with outliers. Other views: box plots of the 6 most affected columns, IsolationForest score histogram (MAT-237)
- headline: "5 outliers in Fare (12.2 %), above 80.97" (one column), "No outliers outside the IQR fences in Age (fences …)", "3 columns with outliers; most: Fare (12.2 %)" (several)
- text: the course's remove / clip / keep / transform decision table

## Notes
Ops: `ops/outliers.py`. Screens `numeric` semantic columns only (ids and constants excluded); IsolationForest median-fills gaps. Fix with `clip` (bounds fitted on train), `log1p`, or `scale(method=robust)`.

- Display sampling (#288): outlier points are drawn at most `OUTLIER_POINTS_MAX` = 500 per column (metrics `n_points_shown`, `n_points_total`, note naming the most truncated column); `flagged_rows` shows the `FLAGGED_SAMPLE` = 20 most anomalous with the note `Showing the 20 most anomalous of N flagged rows.` Fences, counts and isolation forest use the full frame. Other display caps are tracked in #313.
