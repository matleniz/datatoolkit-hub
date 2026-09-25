# Key `train_test_check` — Train / test check

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/train_test_check.py` (ops: `ops/compare.py`) · tests `tests/keys/test_train_test_check.py`, `tests/ops/test_compare.py`
- **What it answers:** can a model fit on this train table be applied to this test table: same columns and dtypes, similar missing rates, ranges and distributions (drift, relative outliers), no unseen categories, no row / entity leak?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `train` | `SourceSpec` | `{"kind": "csv", "path": <demo_data/train.csv>}` | train table |
| `test` | `SourceSpec` | `{"kind": "csv", "path": <demo_data/test.csv>}` | test table |
| `id_columns` | list[str] \| null | null | entity id columns checked for overlap; null = auto (common columns whose semantic type is `id_like` or `group_id` on either side) |

## Result
- metrics: `n_common`, `n_only_train`, `n_only_test`, `n_dtype_mismatch` (raw count, incl. benign int vs float), `n_issues`, `n_errors`, `n_warnings`, `n_drifted` (columns with a warning-level drift finding)
- tables:
  - `issues` — one record per finding `{severity, check, column, message}`, severity in {error, warning, info}, sorted error → warning → info; check in {schema, missing, numeric, categorical, overlap, drift}
  - `columns` — one record per column of the union (train order, then test-only): `in_train`, `in_test`, `dtype_train/test`, `semantic_train/test`, `pct_missing_train/test/delta` (test − train, points), `min/max_train`, `min/max_test`, `pct_test_out_of_range`, `mean_train/test`, `n_unseen_categories`, `unseen_categories`, `pct_test_rows_unseen`, `n_train_only_categories`, `train_only_categories`
  - `overlap` — one `rows` record (test rows identical to a train row) + one `id` record per id column: `{kind, column, n_test, n_test_in_train, pct_test_in_train, row_counter}` (for ids, counts are distinct ids)
  - `numeric_drift` — one record per common numeric column (ids, row counters excluded): `mean/std/p1/median/p99/min/max` `_train` / `_test`, `smd` (standardized mean difference, pooled std), `ks` (two-sample KS statistic), `psi` (10 bins on train quantiles, epsilon 1e-4), `pct_test_below_train_p1`, `pct_test_above_train_p99`, `pct_test_outside_train_range`, `skipped` (reason when a side has < 2 non-null values)
  - `categorical_drift` — long `{column, tvd, value, pct_train, pct_test, diff}` (diff = test − train, points; `tvd` = total variation distance over all categories; top 20 values listed)
- figures: `% missing train vs test` — grouped bar per common column; `<column>: train vs test` — overlaid normalized histograms (same bins) for the top 6 numeric columns by PSI

## Severity choices
- **error**: real dtype conflict on a common column (numeric vs string, datetime or bool vs other); test ids found in train (entity leak) unless the column is a row counter; a requested `id_columns` entry missing on one side.
- **warning**: column only in test (unusable by a model fit on train); semantic-type mismatch; unseen test categories; |missing delta| ≥ 5 points; > 5 % of test values outside the train range; test rows identical to train rows; drift: PSI ≥ 0.25, |SMD| ≥ 0.5, KS ≥ 0.2, ≥ 5 % of test outside train p1–p99, TVD ≥ 0.2.
- **info**: int vs float dtype (numeric on both sides, typically NaN on one side); column only in train (probable target); common columns in a different order; train categories absent from test; 1 ≤ |missing delta| < 5 points; 0 < out-of-range ≤ 5 %; row-counter id overlap; drift: PSI in [0.1, 0.25), 3–5 % of test outside train p1–p99 (~2 % fall outside by construction without drift, hence 3 %).

## Notes
- Numeric range / mean only when both sides are numeric (non-bool) and the column is not an id; a dtype conflict (demo `Age` float vs str because of "unknown") is reported by the schema check instead.
- Categorical checks run when either side is `categorical` / `boolean`; values compared as strings so `1` vs `"1"` is not a new category.
- Row counter: integer column whose distinct values are exactly 0..n-1 or 1..n on **both** sides (index reset in test) → overlap reported as info, not leak.
- Row overlap hashes the stringified common columns **minus** id columns (given or auto) and row counters, so an identical row with a different `Index` / `patient_id` is found. NaN equals NaN in this comparison (a `merge` on the same columns finds fewer). Rows differing only by dtype (14.0 vs "14") do not match. On sparse data, identical rows can be coincidences between different entities (real data: 23 rows, 13 distinct test patients, 5–6 non-null features each).
- Thresholds are module constants in `ops/compare.py` (`MISSING_DELTA_WARNING`, `MISSING_DELTA_INFO`, `OUT_OF_RANGE_WARNING`, `PSI_WARNING`, `PSI_INFO`, `SMD_WARNING`, `KS_WARNING`, `OUTSIDE_P1_P99_WARNING`, `OUTSIDE_P1_P99_INFO`, `TVD_WARNING`), not params. Drift = one finding per column, worst severity wins.
- Drift statistics miss a single imputed spike near the mean (real data: `age_at_diagnosis` = 56.3 on 1 162 test rows, PSI 0.025); the missing-rate check catches it. Real data otherwise: no drift (PSI ≤ 0.025, KS ≤ 0.05, |SMD| ≤ 0.04), ~4–5 s.
- Default demo: `Survived` only in train (info), `Age` float64 vs str (error) + numeric vs text (warning) + 14.63 % → 0 % missing (warning), `Embarked="Q"` unseen (warning, 30 % of test rows), `Fare` 10 % out of range (warning). Auto ids: `PassengerId`; no overlap.
