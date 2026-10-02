# Key `label_join_preview` — Label join preview

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/label_join_preview.py` · test `tests/keys/test_label_join_preview.py`
- **What it answers:** Before joining labels (y) onto X, which join works: by order or by which key, and what would each do to the rows?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `x` | SourceSpec | demo `train.csv` without `Survived` | Features table |
| `y` | SourceSpec | demo `train.csv`, columns `PassengerId`, `Survived` | Labels table |
| `key_columns` | list[str] \| null | null | Candidate key columns; null = auto (columns shared by X and y that are >= 50% distinct on a side, or index-like) |

## Result
- metrics: `x_rows`, `x_columns`, `y_rows`, `y_columns`, `n_key_candidates`, `recommended_mode` (`key` / `order` / `none`), `recommended_key` (empty when none), `n_issues`, `n_errors`, `n_warnings`.
- tables: `candidates` (one row for "by order", one per key: would_join, recommended, uniqueness and duplicate counts per side, match rate X->y and y->X, result_rows, extra_rows from duplicates, aligned), `issues` (severity-ranked: severity, check, candidate, message), `shapes`, `columns` (dtype, n_unique, % missing, shared flag), `head` (first rows of X and y side by side, `X: ` / `y: ` prefixed).
- figures: none.

## Notes
- `would_join` mirrors `ops.join.join_labels`: it is true exactly when the join would not raise (no row loss, duplicate, missing key, column clash, misalignment).
- Recommendation order: a clean key join, else a clean by-order join, else `none` (the best partial key is named in `recommended_key`). Issues on failing candidates are warnings while some candidate works, errors when none does.
- By order, alignment is only verifiable when a shared id / index column exists; otherwise an `info` issue says so.
- Pure diagnostics live in `ops/join.py` (`key_candidates`, `key_diagnostics`, `order_diagnostics`); the join itself is unchanged. Notebook door: `api.label_join_preview(x, y, key_columns=None)`.
