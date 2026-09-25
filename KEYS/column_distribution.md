# Key `column_distribution` — Column distribution

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/column_distribution.py` (ops `ops/distribution.py`, `ops/columns.py`) · test `tests/keys/test_column_distribution.py`
- **Notebook:** `api.distribution(train, test=None, columns=…, target=…, by_label=…)`
- **What it answers:** What do the values of these columns look like, and do they differ between train and test or between label classes?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | frame described |
| `columns` | list[str] (column selector) | `[]` | empty = every numeric / categorical / boolean column (ids, text, dates only when picked), first 20 |
| `compare` | `none` \| `train_vs_test` | `none` | overlay `test` on `source` |
| `test` | SourceSpec | demo test.csv | read only when `compare = train_vs_test` |
| `by_label` | bool | false | split by the classes of `target` (quantile bins for a numeric target) |
| `target` | str \| null (column selector) | null | label column, needed by `by_label` |
| `bins` | int 2–200 | 30 | histogram bins |
| `top_k` | int 1–100 | 10 | categories kept per column, rest `(other)` |
| `target_bins` | int 2–20 | 4 | quantile bins of a numeric target |

## Result
- metrics: `n_columns`, `n_numeric`, `n_categorical`, `n_columns_capped`, `n_groups`, `compare`, `by_label`, `n_missing_in_test`
- tables: `columns` (column, kind), `groups` (group, n_rows), `numeric_summary` + `histograms` (group `numeric`; long format column / group / bin_left / bin_right / count / share), `value_counts` (group `categorical`; column / group / value / count / pct)
- figures: one per column (overlaid share histogram or grouped bar)

## Notes
Bins are shared across groups so shares compare. Groups are `train / <class>`, `test`…; a frame without the target (typical test) stays one group. Numeric vs categorical follows the semantic type (a 1–3 int class is categorical); `(missing)` is its own value.
