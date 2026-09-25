# Key `target_analysis` — Target analysis

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/target_analysis.py` (ops `ops/target.py`) · test `tests/keys/test_target_analysis.py`
- **Notebook:** `api.target_analysis(train, target=…)`
- **What it answers:** Which features relate to the label, and how?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | labeled frame |
| `target` | str (column selector) | `Survived` | label column |
| `columns` | list[str] (column selector) | `[]` | empty = eligible columns but the target, first 30 |
| `task` | `auto` \| `classification` \| `regression` | `auto` | same inference as `feature_selection` |
| `top_k` | int 1–100 | 10 | categories per feature |
| `bins` | int 2–50 | 10 | quantile bins of a numeric feature (regression) |
| `random_state` | int | 0 | mutual information seed |

## Result
- metrics: `task`, `n_rows`, `n_unlabeled`, `n_features`, `n_columns_capped`, `top_feature`, plus `n_classes`, `minority_pct` (classification) or `target_mean`, `target_std` (regression)
- tables: `ranking` (rank, column, kind, mutual_info, association, measure, f_score, f_pvalue, pearson, spearman, pct_missing); classification: `class_balance`, `numeric_by_class`, `class_rate_by_category`; regression: `binned_target_mean`, `target_mean_by_category`
- figures: MI bar + per top-6 feature (box by class drawn from quartiles, class-rate bar, binned-mean line)

## Notes
Rows with a missing target are ignored. Numeric features reuse `ops.selection.filter_scores` (median fill, same MI / F as `feature_selection`); categorical: discrete MI on top-k labels, missing as a label. `association` is 0–1: eta (numeric vs class), Cramér's V (categorical vs class), |Spearman| (numeric vs y), eta of y by category (categorical vs y). MI of continuous and discrete features is estimated differently: compare within a kind first. Errors → `KeyParamsError`.
