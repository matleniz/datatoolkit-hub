# Key `missing_values` — Missing values

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/missing_values.py` · tests `tests/keys/test_missing_values.py`, `tests/ops/test_missing_ops.py`
- **Notebook:** `api.missing(…)`
- **What it answers:** How much is missing, where, is it hidden behind sentinels, does it come in blocks, and was the test set imputed with a constant?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | table to check |
| `test` | SourceSpec \| null | null | optional test source: enables the test imputation-spike check |
| `target` | str \| null | null | target column: counts rows missing it (drop them first) |

## Result
- metrics: `n_rows`, `n_columns`, `n_columns_with_missing`, `n_columns_drop_candidates`, `n_rows_with_missing`, `n_spike_bins`, `n_sentinel_columns`, `n_cooccurring_pairs`, optional `n_rows_missing_target`, `n_test_spikes`
- tables: `missing_rates` (with advice: drop ≥ 60 %, indicator ≥ 20 %), `missing_per_row` (histogram + `spike` flag), `sentinels`, `cooccurrence_pairs` (Jaccard ≥ 0.5), `test_value_spikes`
- figures: missing rate per column, missing fields per row, co-occurrence heatmap

## Notes
Ops: `ops/missing.py` (thresholds are module constants). Sentinels: -9999 … -1, 999, 9999, implausible 0 (only in continuous-looking columns where 0 dominates), 1900-01-01 / 1970-01-01, "unknown", "N/A", "-", "". Test spike: a value ≥ 10 rows, ≥ 1 % of test and ≥ 10× more frequent than in train (real case: `age_at_diagnosis` = 56.3). Fix with `replace_sentinels`, `drop_missing_target`, `impute*`. Nested / binary object columns are not scanned for sentinels.
