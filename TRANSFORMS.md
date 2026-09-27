# Transform ops — catalog

Every op is a workspace step (`target: train | test | both`), an sklearn
`DtkTransformer(op, **params)` and `api.transform(df, op, **params)`. Protocol
and replay: `ARCHITECTURE.md` → Workspace. Recipe: `HOWTO/add-a-transform.md`.
Exact params (types, defaults, descriptions): `transform_schema(op)`.

**Fitted** = learns a state on train (`both` step: train statistics applied to
test, never refitted on test).

## `ops/transforms/cleaning.py`

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `align_to_train` | yes | Realign a shifted / broken numeric column onto train's distribution. | `columns`, `mode`, `group`, `min_rows`, `on_small`, `n_quantiles` |
| `cast` | — | Cast columns to the given dtypes; a failing conversion raises. | `dtypes` |
| `clip` | yes | Clip columns to percentile bounds learned on train (state: bounds). | `columns`, `lower`, `upper` |
| `drop_columns` | — | Remove the listed columns. | `columns`, `missing_ok` |
| `drop_duplicates` | — | Drop duplicate rows, keeping first/last by an explicit sort order. | `subset`, `keep`, `sort_by` |
| `drop_missing_target` | yes | Drop rows whose target is missing (state: rows dropped at fit). | `target` |
| `filter_rows` | — | Keep the rows matching the conditions (no free-form expressions). | `conditions`, `combine` |
| `parse_dates` | — | Parse columns to datetime; unparseable values raise, never become NaT. | `columns`, `format` |
| `rename` | — | Rename columns via an old -> new mapping. | `mapping`, `missing_ok` |
| `replace_sentinels` | — | Turn sentinel values (-999, 'N/A', ...) into NaN, per column. | `sentinels` |
| `standardize_text` | — | Strip / lowercase text columns and map variants to canonical values. | `columns`, `strip`, `lower`, `mapping` |

## `ops/transforms/impute.py`

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `ffill` | — | Carry the last seen value forward in sort_by order (never backward). | `sort_by`, `columns`, `limit` |
| `impute` | yes | Fill missing values with a statistic learned on train (median, mean, mode, constant). | `columns`, `strategy`, `fill_value`, `add_indicator` |
| `impute_iterative` | yes | Fill numeric columns by regressing each one on the others (MICE-style). | `columns`, `max_iter`, `random_state` |
| `impute_knn` | yes | Fill numeric columns from the k nearest train rows (scale features first). | `columns`, `n_neighbors`, `weights` |

## `ops/transforms/encode.py`

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `onehot` | yes | One 0/1 column per train category, in place of the column. | `columns`, `min_frequency`, `drop_first`, `handle_unknown` |
| `ordinal` | — | Code each category by its rank in an explicit order (unknown -> -1). | `categories` |

## `ops/transforms/scale.py`

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `log1p` | — | Replace x by log(1 + x) to tame a right skew (refuses negative values). | `columns` |
| `scale` | yes | Rescale numeric columns with train statistics (standard, minmax, robust, maxabs). | `columns`, `method` |

## `ops/transforms/features.py`

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `bin` | yes | Bin a numeric column by explicit edges or train quantiles. | `column`, `mode`, `edges`, `q`, `labels`, `name` |
| `cyclical` | — | Encode a cyclic column as '<col>_sin' and '<col>_cos'. | `column`, `period` |
| `datetime_parts` | — | Extract hour / dayofweek / month / year / is_weekend from a datetime column. | `column`, `parts` |
| `derive` | — | Add a column combining two columns (ratio, difference, product, days_between). | `a`, `b`, `op`, `name`, `min_denominator` |
| `group_agg` | yes | Join per-group statistics (fitted on train) onto each row. | `group`, `value`, `aggs`, `target` |
| `interactions` | — | Add pairwise products '<a>*<b>' of the listed columns. | `columns`, `interaction_only` |

## `ops/transforms/selection.py` (course 11)

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `drop_low_variance` | yes | Drop numeric columns whose train variance is at or below a threshold. | `threshold`, `columns`, `target` |
| `drop_correlated` | yes (target-aware) | Drop one column of each highly correlated pair (keeps the more target-correlated). | `threshold`, `columns`, `target` |
| `select_k_best` | yes (supervised) | Filter: keep the k best columns by mutual information or F-test with the target. | `target`, `score`, `k`, `percentile`, `task`, `columns`, `random_state` |
| `select_from_model` | yes (supervised) | Embedded: keep the columns an L1 model or a random forest finds important. | `target`, `model`, `threshold`, `max_features`, `task`, `columns`, `random_state` |
| `pca` | yes | Replace numeric columns by principal components `pc1..pcN` (fitted on train). | `n_components`, `columns`, `standardize`, `whiten`, `target`, `prefix` |

State = the `selected` / `dropped` lists (+ scores); `apply` drops exactly
train's `dropped` list, so text columns and the target pass through and test
gets train's columns. Missing values in candidates → error (impute first).
`pca`: `standardize=true` by default (train mean / std in the state),
`n_components` int or variance fraction. **Supervised ops** are registered
with `needs_target=True`: in a workspace the target is a column of the train
frame; in sklearn, `DtkTransformer.fit(X, y)` joins `y` under the `target`
name for fit only.

## `ops/transforms/formula.py` (MAT-128)

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `formula` | yes (variables) | New / replaced column from an expression over columns, numbers, `@variables` and log / log1p / exp / sqrt / abs / round / min / max. | `name`, `expr`, `variables` |

The only free-form expression in the toolkit (approved 2026-09-27): parsed with
Python `ast` against a node whitelist, never `eval`. `variables` =
`[{name, stat, column}]`, stats fitted on train and frozen for test. Missing in
→ missing out; division by ~0 → NaN; `round(x, n)` needs an integer constant `n`.

## Design notes

- `impute_knn` / `impute_iterative`: a fitted KNN / iterative imputer is not
  JSON, so the state is the **train matrix** of the imputed columns; `apply`
  refits the sklearn imputer on it deterministically (fixed `random_state`),
  parity-tested with sklearn. Cost: state = rows × columns, refit at each apply;
  export writes any state whose JSON exceeds 64 KiB (`INLINE_STATE_BYTES`) to
  `states/step_<i>_<op>.json`, referenced from the manifest with its sha256
  (`workspace.export.load_state` reads either form).
- `impute(add_indicator=true)` always emits `<col>_was_missing`, so train and
  test get the same columns. Constant fill defaults to `"MISSING"` for text, 0
  for numbers.
- `onehot`: stable names `<col>_<cat>`, `min_frequency` → `<col>_infrequent`,
  unknown test category → all-zero row (or `handle_unknown=error`).
- `ordinal`: explicit order required (no alphabetical default); unknown → -1.
- `scale`: no clipping beyond the train range (a test value above train max
  maps above 1 with minmax — on purpose); constant column → scale 1.
- `clip` bounds and `bin(mode=qcut)` edges are fitted on train.
- `group_agg` refuses to aggregate the declared `target` (leak); unseen groups → NaN.
- `derive(days_between)` and `datetime_parts` coerce unparseable dates to NaT —
  run `parse_dates` first (it raises) if the column may be dirty.
- `align_to_train` (MAT-58): modes `shift_mean`, `shift_median`, `standardize`
  (alias `standardize_to_train`), `robust` (median + IQR), `quantile` (frame
  quantile → train quantile function, `n_quantiles` = 101). Train statistics
  are the state; the frame's own are measured at apply, so on train it is the
  identity. `group`: per-group train statistics, unseen / small groups fall
  back to global. `min_rows` (30): fewer non-null values → column left
  unchanged (`on_small="raise"` raises). Zero std / IQR → shift only.
- `workspace_pipeline` keeps only `both` steps; row-dropping ops
  (`drop_duplicates`, `filter_rows`, `drop_missing_target`) break X / y
  alignment inside an sklearn `Pipeline` — use them as `train` / `test` steps.
