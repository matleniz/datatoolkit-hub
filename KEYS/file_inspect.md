# Key `file_inspect` — File inspect

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/file_inspect.py` · tests `tests/keys/test_file_inspect.py`
- **Notebook:** contract only (`run_key("file_inspect", {"path": …})`)
- **What it answers:** What does the raw file look like before loading it (bytes, BOM, encoding, line endings, delimiter, header line, empty `Unnamed:` columns; Excel sheets)?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `path` | str | demo train.csv | local path to the file |

## Result
- metrics: `size_bytes`, `modified` (UTC ISO), `first_bytes` (repr of the first bytes), and for text files `bom`, `encoding_guess` (utf-8 else cp1252), `line_endings`, `delimiter`, likely header line, empty `Unnamed:` columns
- tables: `sheets` (sheet, rows, cols) for Excel

## Notes
Course: "look at the file before you load it". Pair with the csv source options (`na_values`, `dtype`, `parse_dates`, `encoding`, `header`).
