# Key `inconsistencies` — Inconsistencies

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/inconsistencies.py` · tests `tests/keys/test_quality_keys.py`, `tests/ops/test_consistency.py`
- **Notebook:** `api.inconsistencies(…)`
- **What it answers:** Is the same fact written several ways in a column (casing / whitespace variants, numbers mixed with strings, ambiguous dd/mm vs mm/dd dates)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | table to check |
| `columns` | list[str] \| null | null | columns to check; null = all text columns |

## Result
- metrics: `columns_checked`, `n_columns_with_variants`, `n_merged_variants`, `n_mixed_type_columns`, `n_ambiguous_date_columns`
- tables: `variants` (distinct before / after normalisation), `suggested mapping` (variant → canonical = most frequent form), `mixed types`, `ambiguous dates`

## Notes
Ops: `ops/consistency.py`. Normalisation = strip, collapse inner whitespace, lower-case. The mapping table plugs directly into the `standardize_text` transform (`mapping`). Ambiguous dates are only reported when most values look like d/m/y. Units cannot be detected (course: use range checks).
