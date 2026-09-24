# Key `dataset_overview` — Dataset overview

- **Category:** analysis
- **Code:** `packages/engine/src/dtk_engine/keys/dataset_overview.py` · test `tests/keys/test_dataset_overview.py`
- **What it answers:** what does this table look like: size, memory, missing values, duplicates, and the type / cardinality of each column?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | `SourceSpec` (discriminated on `kind`; today only `csv`) | `{"kind": "csv", "path": <demo_data/train.csv>}` | table to load. `csv`: `path`, `sep` (`"auto"` sniffs), `encoding` (`utf-8`), `decimal` (`.`), `header` (`0`, null = no header) |
| `head_rows` | int, 1..1000 | 5 | rows returned in the `head` table |

## Result
- metrics: `rows`, `cols`, `memory_mb` (deep, 3 decimals), `pct_missing_cells` (0..100), `n_duplicate_rows` (exact duplicate rows)
- tables: `columns` — one record per column `{column, dtype, semantic_type, n_missing, pct_missing, n_unique, sample_values}` (from `ops.profile.column_profile`); `head` — first `head_rows` rows
- figures: `% missing per column` — Plotly bar of `pct_missing` per column

## Notes
- `semantic_type` (`ops/profile.py`) is a heuristic in {numeric, categorical, boolean, datetime, text, id_like, constant}: ≤1 distinct non-null → constant; numeric {0,1} → boolean; integer or whitespace-free string with distinct/non-null ≥ 0.95 → id_like; strings fully parseable as ISO-8601 → datetime; strings with distinct ratio > 0.5 → text, else categorical. Low-cardinality integers (e.g. `Pclass`) stay numeric.
- Known weak spots (review of PR #2): a sparse column with few, all-distinct values (demo `Cabin`) or any all-distinct integer feature → `id_like`; numbers polluted by a token (`Age` = "unknown" in demo test) → `text`; year strings → `datetime`.
- `sep="auto"` uses pandas' python engine on the whole file: slow on big CSVs.
- A missing or unreadable file raises `SourceError` (not `KeyParamsError`); an unknown `kind` or misspelled param raises `KeyParamsError`.
- Default demo train: 41 rows × 12 cols, 1 deliberate duplicate row.
