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
- Scores (mutual information, RF importance, L1, RFECV) use at most 10 000 labeled rows (`SCORE_SAMPLE_SIZE`, `ops/selection.py`): a seeded sample, stratified on a classification target (plain draw for regression or a class under 2 rows); metric `n_scored_rows` and a text note when sampled; variance and `pct_missing` still use every row. Parkinson train (55 603 rows): ~67 s → ~4.5 s (datatoolkit-issues#77).
- Scoring median-fills missing feature values (said in the text); the selection **ops** refuse missing values — impute first.
- PCA here is on standardized numeric features. L1 uses `l1_ratio=1.0` (sklearn 1.9 deprecates `penalty=`).
- Errors → `KeyParamsError`: target not in the frame or all missing, `task=regression` with a non-numeric target, explicit `columns` absent / non-numeric / the target.
- No numeric feature besides the target (default `columns`): no exception; the Result has metrics `task, n_rows, n_features=0, n_non_numeric_columns, n_encode_steps`, tables `non_numeric_columns`, `encode_first_steps` (preprocessing_advisor recommendations) and `families`, no figures; the text says to apply those steps and re-run.
- Apply the choice with the selection ops (`TRANSFORMS.md` → `selection.py`), fitted on train.
