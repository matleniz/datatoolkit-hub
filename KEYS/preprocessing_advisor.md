# Key `preprocessing_advisor` — Preprocessing advisor

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/preprocessing_advisor.py` (logic `ops/advisor.py`) · tests `tests/keys/test_preprocessing_advisor.py`, `tests/ops/test_advisor.py`
- **Notebook:** `api.advise(train_df, test_df, model_family="linear", target="y")`
- **What it answers:** what should I do to each column before modelling — as workspace steps I can apply?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | train data |
| `test` | SourceSpec \| null | null | enables the train-only (leak) check, unseen categories, test-side sentinels / types |
| `model_family` | `tree` \| `linear` \| `distance` \| `neural` \| null | null | decides scaling, log1p and high-cardinality encoding; null = advice for every family |
| `target` | str \| null | null | excluded from features; rows missing it dropped; target-derived columns flagged |

## Result
- metrics: `model_family`, `n_columns`, `n_recommendations`, `n_warnings`, `n_leak_warnings`, `n_columns_dropped`, `onehot_columns_produced`
- tables: `recommendations` (order, column, category, severity, advice, op, target, params — each row is a workspace step; `ops.advisor.as_steps` lists them), `columns` (semantic_type, pct_missing, n_unique, skew, pct_outliers_iqr, onehot_columns, action)
- text: the warnings

## Notes
- Order: rows (`drop_missing_target`, exact `drop_duplicates`, on train) → drop / leak (train-only, id_like, a column named after the target, |corr| ≥ 0.95 with the target, ≥ 60 % missing, constant, group_id → use as GroupKFold groups, free text, nested (lists / dicts per cell: flatten with json_normalize / explode first) and binary (bytes: decode first) as warnings; test-only columns dropped on test) → `replace_sentinels` → `standardize_text` → `cast` float64 (numbers stored as text) → `impute` (median numeric, most_frequent categorical, "MISSING" constant at ≥ 20 %; indicator for numeric at ≥ 20 %) → `log1p` (skew > 1, no negative, not for trees) → `scale` (not for trees; robust when > 1 % IQR outliers after log1p) → encoding (ordinal hint from known vocabularies like low / medium / high; binary → onehot `drop_first` unless test has unseen values; `min_frequency=10` when > 10 categories with rare ones; > 50 categories → warning, or an arbitrary ordinal code for trees; datetime → `datetime_parts` then drop; bool → cast int64).
- Cleaning recommendations are applied with the real ops to working copies before later checks.
- Numeric sentinels (-1, 999…) count only outside the range of the other values; the "implausible 0" heuristic is reported by `missing_values` but never auto-replaced.
- Tests: every recommendation parses as a valid step for every family; replaying the whole plan leaves no NaN, only numeric features, same columns on train and test. On the demo data it finds the duplicate row, the `PassengerId` leak, `Cabin` 88 % missing, `Age` as text with "unknown", `Embarked="Q"` unseen in train.
