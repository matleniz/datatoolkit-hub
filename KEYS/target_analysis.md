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
| `bins` | int 2–50 | 10 | quantile bins of a numeric feature (class counts and mean target per bin) |
| `target_bins` | int 2–50 | 10 | histogram bins for `target_histogram`, when the target is numeric (regression, MAT-174) |
| `random_state` | int | 0 | mutual information seed |

## Result
- metrics: `task`, `n_rows`, `n_unlabeled`, `n_features`, `n_columns_capped`, `top_feature`, `focus_feature` (first of `columns` if given, else `top_feature`), plus `n_classes`, `minority_pct` (classification) or `target_mean`, `target_std` (regression)
- tables: `ranking` (rank, column, kind, mutual_info, association, measure, f_score, f_pvalue, pearson, spearman, pct_missing); classification: `class_balance`, `numeric_by_class`, `class_rate_by_category`, `class_counts_by_category` (column, value, class, count, pct_of_value), `class_counts_by_bin` (column, bin, class, count, pct_of_bin) — long format, readable bin labels ("11.7–13.5"); regression: `binned_target_mean`, `target_mean_by_category` (both with `std_target`, `q1_target`, `q3_target`), `target_histogram` (regression: bin edges + counts of the target itself, MAT-174)
- figures: only for the focused feature (MAT-238). Classification: **main** = grouped bars of row counts per class by category / quantile bin, + stacked 100 % share view, + box by class for a numeric feature. Regression: **main** = mean-target curve with an IQR band (rows per point in hover). Always: "Association with the target" = horizontal MI bars, sorted
- headline: about the focused feature, ignoring bins / categories smaller than max(5 rows, 5 %): "Survived rate ranges from 29 % (male) to 54 % (female) across Sex", or "… varies across Fare bins (small sample)" when too few remain

## Notes
Rows with a missing target are ignored. Numeric features reuse `ops.selection.filter_scores` (median fill, same MI / F as `feature_selection`); categorical: discrete MI on top-k labels, missing as a label. `association` is 0–1: eta (numeric vs class), Cramér's V (categorical vs class), |Spearman| (numeric vs y), eta of y by category (categorical vs y). MI of continuous and discrete features is estimated differently: compare within a kind first. Errors → `KeyParamsError`.
