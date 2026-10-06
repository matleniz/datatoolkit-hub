# Key `impute_benchmark` — Impute benchmark

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/impute_benchmark.py` (ops `src/dtk_engine/ops/impute_benchmark.py`) · test `tests/keys/test_impute_benchmark.py`
- **What it answers:** which `impute` strategy recovers a numeric column best?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | source | demo train CSV | data to study (the agent fills it from the UI context) |
| `columns` | list[str] (numeric) | `["Age"]` | one study per column |
| `strategies` | list | `[median, mean]` | any of `median`, `mean`, `group_mean`, `group_prev`, `group_interp` |
| `by` | str? | — | entity column, needed by the group strategies |
| `order` | str? | — | needed by `group_prev` / `group_interp` |
| `mask_fraction` | float in ]0,1[ | 0.2 | share of each column's known values hidden then re-imputed |
| `seed` | int | 0 | masking seed (deterministic) |

## Result
- metrics: `n_columns`, `n_strategies`, `mask_fraction`, `seed`, `best_<column>` (lowest-RMSE strategy)
- tables: `benchmark` (column, strategy, n_masked, n_filled, coverage, rmse, mae, median_abs_error)
- figures: none; headline "Best by RMSE: …"

## Notes
Per column, a seeded share of the KNOWN values is masked (at least one, never
all); every strategy is scored on the same masked rows by running the existing
`impute` transform on the masked frame (one implementation, nothing leaks into
learned statistics, no fallback fill). Coverage = filled / masked is reported
apart; errors are on filled cells only, so compare RMSE at similar coverage.
Not covered: per-cohort breakdown, knn / iterative / formula strategies.
Born from the parkison agent session (datatoolkit-issues#159): on its train
set, `ledd` → `group_interp` best (RMSE ≈ 34).
