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
| `cast` | — | Cast columns to the given dtypes; a failing conversion raises. | `dtypes` |
| `clip` | yes | Clip columns to percentile bounds learned on train (state: bounds). | `columns`, `lower`, `upper` |
| `drop_columns` | — | Remove the listed columns. | `columns`, `missing_ok` |
| `drop_duplicates` | — | Drop duplicate rows, keeping first/last by an explicit sort order. | `subset`, `keep`, `sort_by` |
| `drop_high_missing` | yes | Drop columns whose train missing fraction exceeds a threshold (state: dropped); same columns dropped on train and test; never drops `target`. | `threshold`, `exclude`, `target` |
| `drop_missing_target` | yes | Drop rows whose target is missing (state: rows dropped at fit). | `target` |
| `extract` | — | Pull named regex groups from a text column into new typed columns (numeric-looking groups cast to float); pattern length capped at 256 (stdlib `re`, no match timeout). | `column`, `pattern`, `prefix`, `errors` |
| `filter_rows` | — | Keep the rows matching the conditions (no free-form expressions). | `conditions`, `combine` |
| `parse_dates` | — | Parse columns to datetime; unparseable values raise, never become NaT. | `columns`, `format` |
| `reorder_columns` | — | Move the listed columns (kept in the given order) to the start, the end, or just before / after an anchor column; the other columns keep their relative order. Stateless; never a full explicit order, so it survives columns added upstream. | `columns`, `position`, `anchor`, `missing_ok` |
| `rename` | — | Rename columns via an old -> new mapping. | `mapping`, `missing_ok` |
| `replace_sentinels` | — | Turn sentinel values (-999, 'N/A', ...) into NaN, per column. | `sentinels` |
| `standardize_text` | — | Strip / lowercase text columns, collapse separators (`-`/`_`/`.`/repeated whitespace) into a single space, and map variants to canonical values. | `columns`, `strip`, `lower`, `unify_separators`, `mapping` |
| `to_numeric` | — | Parse text numbers (currency symbols/codes, thousands / decimal separators, percent signs) to float. | `columns`, `decimal`, `thousands`, `percent`, `errors` |

## `ops/transforms/align.py`

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `align_to_train` | yes | Realign a shifted / broken numeric column onto train's distribution. | `columns`, `mode`, `group`, `min_rows`, `on_small`, `n_quantiles` |

## `ops/transforms/impute.py`

| Op | Fitted | What it does | Params |
|---|---|---|---|
| `ffill` | — | Carry the last seen value forward in sort_by order (never backward); with `by`, only within each entity. | `sort_by`, `columns`, `by`, `limit` |
| `impute` | yes | Fill missing values with a statistic learned on train (median, mean, mode, constant), a formula of other columns (`formula`), or within-entity fills (`group_mean`, `group_prev`, `group_interp`). | `columns`, `strategy`, `fill_value`, `expr`, `by`, `order`, `fallback`, `add_indicator` |
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
| `group_agg` | yes | Join per-group statistics (fitted on train) onto each row: `mean` / `std` / `count` / `median` / `min` / `max`, and `first` / `last` (first / last observed value in `order`; rows with a missing `order` ignored). | `group`, `value`, `aggs`, `order`, `target` |
| `interactions` | — | Add pairwise products '<a>*<b>' of the listed columns. Superseded by `polynomial` for new steps (kept for backward compat of saved workspaces). | `columns`, `interaction_only` |
| `polynomial` | yes | Replace numeric columns by a `PolynomialFeatures` expansion; readable names (`a^2`, `a*b`); refuses when output width exceeds `max_output_columns`. | `columns`, `degree`, `interaction_only`, `include_bias`, `max_output_columns` |
| `power_transform` | yes | Power-map columns in place (Yeo-Johnson / Box-Cox), fitted on train. | `columns`, `method`, `standardize` |
| `quantile_transform` | yes | Map columns to a uniform / normal distribution via train quantiles. | `columns`, `output_distribution`, `n_quantiles`, `random_state` |
| `spline` | yes | Replace columns by B-spline bases using train knot positions. | `columns`, `n_knots`, `degree`, `knots`, `include_bias`, `max_output_columns` |

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
| `formula` | yes (variables) | New / replaced column from an expression over columns, numbers, `@variables`, the constant `pi`, and log / log1p / log2 / log10 / exp / sqrt / abs / round / min / max / sin / cos / tanh / floor / ceil / sign / square / clip / where / isnull, and the NaN-aware isna / notna (0/1, never NaN) / fillna(x, v) (comparisons → 0/1; `where(c, a, b)` returns NaN when `c` is NaN, so test missingness first, e.g. `where(isna(x), y - 5.6, x)`), and the within-entity `group_mean(x, by=col)`, `group_prev(x, by=col, order=expr)`, `group_interp(x, by=col, order=expr)` (computed per frame; the only keyword arguments; `by=` names a column). | `name`, `expr`, `variables` |

The only free-form expression in the toolkit (approved 2026-09-27): parsed with
Python `ast` against a node whitelist, never `eval`. `variables` =
`[{name, stat, column}]`, stats fitted on train and frozen for test. Missing in
→ missing out; division by ~0 → NaN; `round(x, n)` needs an integer constant `n`.
`where(cond, a, b)` and comparisons (`<`, `<=`, `>`, `>=`, `==`, `!=`) were
added in MAT-173 alongside `clip`/`floor`/`ceil`/`sign`/`square`/`tanh`/
`log2`/`log10`/`isnull`, still no `eval`.

**Python-style one-liners** (MAT-241, validated 2026-09-30): each construct is
rewritten onto the same whitelist, nothing is exec'd. Accepted: `np.<f>` /
`numpy.<f>` for the whitelisted functions (+ `np.pi`, `np.isnan` → isnull,
`np.minimum` / `np.maximum`); `a if cond else b` (→ where, nestable); `and` /
`or` / `not` and `&` / `|` / `~` element-wise; chained comparisons
(`18 <= Age < 65`); `**`, `%`, `//`; builtins `abs`, `round`, `min(a, b)`,
`max(a, b)`; `df["Col name"]` / `df.col` for column names that are not
identifiers. Refused with a message naming the construct: lambda, imports,
comprehensions, method calls (`.apply`), calling a call's result, any dunder,
attribute access other than `np.<f>` / `df.<col>`, subscript other than
`df["col"]`, unlisted functions (`eval`, `open`, `getattr`, `np.random`,
`np.load`, `torch.*`). No torch / arbitrary Python: formulas are replayed and
exported as sklearn pipelines, so they stay a safe, deterministic subset.

## Design notes

- `impute_knn` / `impute_iterative`: a fitted KNN / iterative imputer is not
  JSON, so the state is the **train matrix** of the imputed columns; `apply`
  refits the sklearn imputer on it deterministically (fixed `random_state`),
  parity-tested with sklearn. Cost: state = rows × columns, refit at each apply;
  export writes any state whose JSON exceeds 64 KiB (`INLINE_STATE_BYTES`) to
  `states/step_<i>_<op>.json`, referenced from the manifest with its sha256
  (`workspace.export.load_state` reads either form).
- `impute(strategy=formula)` (datatoolkit-issues#7): one numeric column, `expr`
  in the `formula` language without `@variables`; stateless (the expression is
  the state), evaluated on each frame and written only into its missing cells;
  a row whose expression is NaN stays missing.
- Group fills (datatoolkit-issues#8, `ops/groups.py`): each frame (train, test)
  is filled from the same entity's rows in that frame, so the rule carries over
  to test without leaking train rows; `impute`'s optional `fallback` (median /
  mean / most_frequent) is fitted on train for cells the entity cannot fill.
  `group_prev` never fills backward; `group_interp` is linear in `order`
  between the entity's neighbours, no extrapolation. Rows with a missing entity
  or order key are neither filled nor used. Vectorised (bincount / groupby
  ffill). "Past rows only" mode: later.
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
- `reorder_columns` (datatoolkit-issues#161): `position` = first | last | before | after;
  `anchor` required for before / after, forbidden otherwise, never one of the moved
  columns (rejected at parse); `columns` must not repeat. `missing_ok` applies to
  `columns` only (as `drop_columns`); a missing `anchor` always raises, so train and
  test cannot silently diverge.
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
- `to_numeric`'s currency detection (advisor + workspace profile
  `currency_as_text`) is a best-effort heuristic in
  `ops/profile.py::numeric_text_format`: guesses `decimal`/`thousands` from the
  separators seen once the currency symbol/code and `%` are stripped (comma
  decimal wins when both `.` and `,` are absent from a value with a space
  group).
- `drop_high_missing` never drops the declared `target`, mirroring
  `group_agg`'s target-leak guard.
- `extract`: `errors=raise` (default) fails on a non-null cell with no match;
  `coerce` fills the group columns with NaN. Invalid pattern / missing named
  groups fail at param validation (`KeyParamsError`). Output names are
  `<prefix>_<group>` (default prefix = source column). The op does not
  compute derived values (e.g. the mean of a range) — chain `derive` /
  `formula` on the extracted `low`/`high` columns.
- `polynomial` / `power_transform` / `quantile_transform` / `spline` raise a
  clear engine error when the selected columns still have missing values
  (impute first); the Studio front surfaces this as a blocking hint before
  preview (MAT-191).
- `ops/consistency.py::normalize` (used by the `inconsistencies` key's variant
  detection and the workspace profile's `variants` field) applies the same
  separator-unification rule as `standardize_text`'s `unify_separators`, so
  `'site-a'` / `'site_a'` / `'Site A'` merge without an extra step.
