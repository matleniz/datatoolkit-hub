# Key `inconsistencies` — Inconsistencies

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/inconsistencies.py` · tests `tests/keys/test_quality_keys.py`, `tests/ops/test_consistency.py`
- **Notebook:** `api.inconsistencies(…)`
- **What it answers:** Is the same fact written several ways in a column (casing / whitespace variants, numbers mixed with strings, ambiguous dd/mm vs mm/dd dates, date columns written in several formats)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train.csv | table to check |
| `columns` | list[str] \| null | null | columns to check; null = all text columns |

## Result
- metrics: `columns_checked`, `n_columns_with_variants`, `n_merged_variants`, `n_mixed_type_columns`, `n_ambiguous_date_columns`, `n_mixed_date_format_columns`
- tables: `variants` (distinct before / after normalisation), `suggested mapping` (variant → canonical = most frequent form), `mixed types`, `ambiguous dates`, `mixed date formats` (`column, n_values, n_date_like, n_formats, formats, n_not_date, examples`)
- text: advice for the mapping and, when dates mix formats, to unify them before `parse_dates`

## Notes
Ops: `ops/consistency.py`. Normalisation = strip, collapse inner whitespace, lower-case. The mapping table plugs directly into the `standardize_text` transform (`mapping`). Ambiguous dates are only reported when most values look like d/m/y. Units cannot be detected (course: use range checks). Mixed date formats: each string is classed as `yyyy-mm-dd` (separators - / ., time part ignored), `dd/mm/yyyy`, `mm/dd/yyyy`, `nn/nn/yyyy` (undecidable; folded into the only order seen in the column), `dd mon yyyy`, `mon dd yyyy`. A column is a date column when ≥ 50 % of its strings match (`DATE_LIKE_MIN_SHARE`); reported when it has ≥ 2 formats or non-date values. Fix: one format, then `parse_dates` (single `format`, raises otherwise); non-dates → NaN with `replace_sentinels`.
