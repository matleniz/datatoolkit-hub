# Key `file_inspect` — File inspect

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/file_inspect.py` · tests `tests/keys/test_file_inspect.py`
- **Notebook:** contract only (`run_key("file_inspect", {"path": …})`)
- **What it answers:** What does the raw file look like before loading it (bytes, BOM, encoding, decimal mark, line endings, delimiter, header line or none, empty `Unnamed:` columns, leading-zero columns, first malformed line; Excel sheets and header row; JSON record path) — and which spec loads it?

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `path` | str | demo train.csv | local path to the file |

## Result
- metrics: `size_bytes`, `modified` (UTC ISO), `first_bytes` (repr of the first bytes), and for text files `bom`, `encoding_guess` (BOM, utf-8, cp1252 or latin-1), `line_endings`, `delimiter`, `decimal_guess`, `header_guess` (`present` \| `none`); when present `header_line` (1-based), `title_lines_above_header`, `unnamed_columns`, `leading_zero_columns`; when none with title lines `title_lines_above_data`; `bad_line` (first malformed line in the 64 KB sample, or `none`); `mixed_separators` (`none`, or the alt delimiter with its line numbers) with `separator_line_counts` when found; `junk_lines` when junk lines above the header fooled the delimiter sniffer; `load_spec` = JSON SourceSpec with the guesses applied, for whichever kind was sniffed, not csv-only → `api.load(json.loads(m["load_spec"]))`. csv: sep, encoding, decimal, header, `dtype: str` for leading-zero columns, `skiprows` only when junk lines fooled the sniffer (ordinary title lines stay `header: N`), `mixed_sep: "normalize"` when mixed separators are found. parquet: `{kind, path}`. excel: `{kind, path, sheet, header}` from the first sheet's `suggested_header` (0-based), plus the `sheets` table. json/jsonl: `{kind, path, lines}` (`lines=true` for jsonl) and optional `record_path` when suggested; JSON files (≤ 50 MB) also get `suggested_record_path` when records sit under an envelope. Parquet / Excel / JSON skip the csv text sniffer (`first_bytes` / delimiter facts are csv-only).
- tables: `sheets` (sheet, rows, cols, `suggested_header` 0-based) for Excel; `record_paths` (record_path, records) for enveloped JSON

## Notes
Header detection honours CSV quoting (quoted separators / newlines); a line where every numeric column is already numeric is data, not a header (UCI dumps). Course: "look at the file before you load it". Pair with the csv source options (`na_values`, `dtype`, `parse_dates`, `encoding`, `header`).
