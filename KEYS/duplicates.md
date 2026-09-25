# Key `duplicates` — Duplicates

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/duplicates.py` · tests `tests/keys/test_quality_keys.py`, `tests/ops/test_duplicates.py`
- **Notebook:** `api.duplicates(…)`
- **What it answers:** Which rows are exact copies, which share an identity key, and where rows with the same key disagree (conflicts)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | table to check |
| `subset` | list[str] \| null | null | identity columns for partial duplicates; null = auto (`id_like` / `group_id` columns) |

## Result
- metrics: `rows`, `n_exact_duplicates`, `pct_exact_duplicates`, `subset`; with a subset also `n_partial_duplicate_rows`, `n_duplicate_groups`, `n_conflict_groups`
- tables: `exact duplicates` (all members, keep=False), `duplicate groups` (numbered `duplicate_group`), `conflicting columns` (column, n_groups, pct_groups), `conflicting rows`
- `duplicate groups` / `conflicting columns` / `conflicting rows` exist only when the subset (explicit or auto) is non-empty; every sample table is capped at `SAMPLE_ROWS = 20` rows

## Notes
Ops: `ops/duplicates.py` (`exact_duplicates`, `duplicate_groups`, `conflicts`). Inspection only; fix with the `drop_duplicates` transform (explicit `sort_by`, keep last = most recent). Unhashable cells (lists, dicts) are compared as strings. Course: Duplicates and Inconsistencies.
