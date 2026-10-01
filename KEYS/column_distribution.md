# Key `column_distribution` — Column distribution

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/column_distribution.py` (ops `ops/distribution.py`, `ops/columns.py`) · test `tests/keys/test_column_distribution.py`
- **Notebook:** `api.distribution(train, test=None, columns=…, target=…, by_label=…, by=…, target_bins=…)`
- **What it answers:** What do the values of these columns look like, and do they differ between train and test or between label classes?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | frame described |
| `columns` | list[str] (column selector) | `[]` | empty = every numeric / categorical / boolean column (ids, text, dates only when picked), first 20 |
| `compare` | `none` \| `train_vs_test` | `none` | overlay `test` on `source` |
| `test` | SourceSpec | demo test.csv | read only when `compare = train_vs_test` |
| `by_label` | bool | false | split by the classes of `target` (quantile bins for a numeric target) |
| `by` | str \| null (column selector) | null | split by any other frame column (top-k + `(other)` or quantile bins); exclusive with `by_label`. Numeric column vs numeric `by`: sampled scatter (≤ 2000 points), `vs_by` table, `pearson` / `spearman` |
| `target` | str \| null (column selector) | null | label column, needed by `by_label` |
| `bins` | `auto` \| int 2–200 | `auto` | histogram bins; `auto` = smart per-column default (MAT-174, see Notes) |
| `bin_edges` | list[float] \| null | null | explicit bin edges, overrides `bins` |
| `range_min_pct`, `range_max_pct` | float 0–100 | 0 / 100 | clip the histogram range to these percentiles of the column (`range_min_pct` must be ≤ `range_max_pct`) |
| `log_x`, `log_y` | bool | false | `log_x`: bin `log1p(x)` (negative values dropped from the histogram); `log_y`: log scale on the count axis of the figure |
| `norm` | `count` \| `density` \| `share` | `share` | histogram height: share of rows, raw count, or density |
| `cumulative` | bool | false | also compute the running total / share |
| `top_k` | int 1–100 | 10 | categories kept per column, rest `(other)` |
| `target_bins` | int 2–20 | 4 | quantile bins of a numeric target |

## Result
- metrics: `n_columns`, `n_numeric`, `n_categorical`, `n_columns_capped`, `n_groups`, `compare`, `by_label` (target name or `none`), `by` (column or `none`), `n_missing_in_test`, plus the echoed options `bins`, `norm`, `cumulative`, `log_x`, `log_y`; `pearson` / `spearman` when a numeric column is split by a numeric `by`
- tables: `columns` (column, kind), `groups` (group, n_rows), `numeric_summary` + `histograms` (group `numeric`; long format column / group / bin_left / bin_right / count / share, plus `density`, `cumulative_count`, `cumulative_share` when requested), `value_counts` (group `categorical`; column / group / value / count / pct), `vs_by` (numeric column split by a numeric `by`)
- figures: one per column (overlaid share histogram or grouped bar); a scatter figure `{col} vs {by}` when both are numeric

## Notes
Bins are shared across groups so shares compare. Groups are `train / <class>`, `test`…; a frame without the target (typical test) stays one group. Numeric vs categorical follows the semantic type (a 1–3 int class is categorical); `(missing)` is its own value.
`bins=auto` and the other smart defaults (log-scale suggestion for heavy right
skew, `top_k` adapted to cardinality) are computed per column by
`ops/suggested.py` (Freedman–Diaconis / Sturges, integer-aligned for discrete
ints) and exposed as `suggested_params` on each column's `column_profiles`
entry (MAT-174) — fronts should read that instead of hard-coding a default.
