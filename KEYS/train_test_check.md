# Key `train_test_check` — Train / test check

- **Category:** analysis
- **Code:** `packages/engine/src/dtk_engine/keys/train_test_check.py` (ops: `ops/compare.py`) · tests `tests/keys/test_train_test_check.py`, `tests/ops/test_compare.py`
- **What it answers:** can a model fit on this train table be applied to this test table: same columns and dtypes, similar missing rates and ranges, no unseen categories, no row / entity leak?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `train` | `SourceSpec` | `{"kind": "csv", "path": <demo_data/train.csv>}` | train table |
| `test` | `SourceSpec` | `{"kind": "csv", "path": <demo_data/test.csv>}` | test table |
| `id_columns` | list[str] \| null | null | entity id columns checked for overlap; null = auto (common columns whose semantic type is `id_like` or `group_id` on either side) |

## Result
- metrics: `n_common`, `n_only_train`, `n_only_test`, `n_dtype_mismatch` (raw count, incl. benign int vs float), `n_issues`, `n_errors`, `n_warnings`
- tables:
  - `issues` — one record per finding `{severity, check, column, message}`, severity in {error, warning, info}, sorted error → warning → info; check in {schema, missing, numeric, categorical, overlap}
  - `columns` — one record per column of the union (train order, then test-only): `in_train`, `in_test`, `dtype_train/test`, `semantic_train/test`, `pct_missing_train/test/delta` (test − train, points), `min/max_train`, `min/max_test`, `pct_test_out_of_range`, `mean_train/test`, `n_unseen_categories`, `unseen_categories`, `pct_test_rows_unseen`, `n_train_only_categories`, `train_only_categories`
  - `overlap` — one `rows` record (test rows identical to a train row) + one `id` record per id column: `{kind, column, n_test, n_test_in_train, pct_test_in_train, row_counter}` (for ids, counts are distinct ids)
- figures: `% missing train vs test` — grouped bar per common column

## Severity choices
- **error**: real dtype conflict on a common column (numeric vs string, datetime or bool vs other); test ids found in train (entity leak) unless the column is a row counter; a requested `id_columns` entry missing on one side.
- **warning**: column only in test (unusable by a model fit on train); semantic-type mismatch; unseen test categories; |missing delta| ≥ 5 points; > 5 % of test values outside the train range; test rows identical to train rows.
- **info**: int vs float dtype (numeric on both sides, typically NaN on one side); column only in train (probable target); common columns in a different order; train categories absent from test; 1 ≤ |missing delta| < 5 points; 0 < out-of-range ≤ 5 %; row-counter id overlap.

## Notes
- Numeric range / mean only when both sides are numeric (non-bool) and the column is not an id; a dtype conflict (demo `Age` float vs str because of "unknown") is reported by the schema check instead.
- Categorical checks run when either side is `categorical` / `boolean`; values compared as strings so `1` vs `"1"` is not a new category.
- Row counter: integer column whose distinct values are exactly 0..n-1 or 1..n on **both** sides (index reset in test) → overlap reported as info, not leak.
- Row overlap hashes the stringified common columns **minus** id columns (given or auto) and row counters, so an identical row with a different `Index` / `patient_id` is found. NaN equals NaN in this comparison (a `merge` on the same columns finds fewer). Rows differing only by dtype (14.0 vs "14") do not match. On sparse data, identical rows can be coincidences between different entities (real data: 23 rows, 13 distinct test patients, 5–6 non-null features each).
- Thresholds are module constants in `ops/compare.py` (`MISSING_DELTA_WARNING`, `MISSING_DELTA_INFO`, `OUT_OF_RANGE_WARNING`), not params.
- Default demo: `Survived` only in train (info), `Age` float64 vs str (error) + numeric vs text (warning) + 14.63 % → 0 % missing (warning), `Embarked="Q"` unseen (warning, 30 % of test rows), `Fare` 10 % out of range (warning). Auto ids: `PassengerId`; no overlap.
