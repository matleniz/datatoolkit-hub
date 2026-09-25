# Key `feature_selection` — Feature selection

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/feature_selection.py` (logic `ops/selection.py`) · tests `tests/keys/test_feature_selection.py`, `tests/ops/test_selection_ops.py`
- **Notebook:** `api.select_features(df, target="y")`
- **What it answers:** which columns carry signal about the target, which are redundant or constant, and how many principal components hold the variance? (course 11)

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | table with the target |
| `target` | str | `Survived` (demo) | target column |
| `task` | `auto` \| `classification` \| `regression` | `auto` | auto = classification for a non-numeric target or few integer values |
| `columns` | list[str] \| null | null | numeric features to score; null = every numeric column but the target |
| `wrapper` | bool | false | also run RFECV (wrapper family: one fit per step, slow) |
| `random_state` | int | 0 | seed of the models |

## Result
- metrics: `task`, `n_rows`, `n_features`, `n_collinear_pairs`, `n_near_constant`, `top_feature`, components for 90 / 95 / 99 % variance
- tables: `feature_scores` (variance, pct_missing, mutual_info, f_score / f_pvalue, abs_corr_target, l1_coef, tree_importance, optional rfecv_rank, combined_rank), `collinear_pairs (|corr| >= 0.9)`, `near_constant` (top value ≥ 95 %), `pca_explained_variance`, `families` (filter / wrapper / embedded + blind spots), `suggested_steps` (valid workspace steps)
- figures: feature scores, PCA cumulative explained variance

## Notes
- Scoring median-fills missing feature values (said in the text); the selection **ops** refuse missing values — impute first.
- PCA here is on standardized numeric features. L1 uses `l1_ratio=1.0` (sklearn 1.9 deprecates `penalty=`).
- Apply the choice with the selection ops (`TRANSFORMS.md` → `selection.py`), fitted on train.
