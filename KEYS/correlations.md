# Key `correlations` — Correlations

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/correlations.py` (ops `ops/correlation.py`) · test `tests/keys/test_correlations.py`
- **Notebook:** `api.correlations(train, method="spearman")`
- **What it answers:** Which numeric columns move together (redundant features)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | frame |
| `columns` | list[str] (column selector, numeric) | `[]` | empty = every numeric column but ids and the target, first 50 |
| `method` | `pearson` \| `spearman` \| `kendall` | `pearson` | spearman = monotonic, on ranks; kendall = rank concordance, robust to outliers, slower (MAT-174) |
| `threshold` | float (0, 1] | 0.9 | pairs with \|corr\| ≥ this |
| `target` | str \| null (column selector) | null | excluded from the matrix; decides which column of a pair to keep |

## Result
- metrics: `method`, `n_columns`, `n_columns_capped`, `n_pairs`, `threshold`, `max_abs_corr`, `n_undefined`
- tables: `correlated_pairs (|corr| >= t)` (a, b, corr, abs_corr, n_rows, keep), `matrix`, `abs_corr_with_target` (with a target), `suggested_steps` (one `drop_correlated` both-step, pearson only — appliable from the front)
- figures: heatmap (values annotated up to 20 columns)

## Notes
Pairwise-complete correlations; constant columns reported as undefined. `keep` follows the `drop_correlated` rule (more target-correlated kept, else first in column order); the suggested step is tested to drop the same column. Errors → `KeyParamsError`.
