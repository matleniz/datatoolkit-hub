# 2026-10-15 — first-time-user E2E of the engine (night mandate)

Worker `night-e2e` (sonnet) followed only the engine README from a fresh venv
and a fresh `DTK_HOME`, on a synthetic non-demo dataset, through the three
doors (Python, HTTP `dtk-api`, MCP `dtk-mcp` over stdio). Baseline the same
night (scout `night-scout`): main abae4fe, ruff clean, pytest 1620 passed /
1 skipped. Scout issues: #223-#226; E2E issues: #227-#233.

**Environment**
- Linux 6.6 (WSL2), Python 3.12.3 system, venv on Python 3.14, uv 0.11.8.
- Engine installed from `git+https://github.com/matleniz/datatoolkit` at sha abae4fe (= origin/main).
- Fresh venv and fresh `DTK_HOME` in `/tmp/tmp.OaDAbUE6To`.
- Dataset: 3050 rows, 6 columns (numeric and categorical, 300 missing ages, 200 missing cities, 50 duplicate rows, target `churn`).
- API server on a free port, stopped by PID. Nothing is left running.
**Steps**
1. Install with the README command (`uv pip install git+...`): OK, 2 s.
2. First command after install: FAIL.
- `dtk-api --help` fails with `ModuleNotFoundError: fastapi`.
- `dtk-mcp --help` fails with `ModuleNotFoundError: anyio`.
- This is #224 (comment added).
3. Extras: CONFUSING.
- The README's `uv sync --extra api` gives "No pyproject.toml found" outside a clone.
- What works is `uv pip install "dtk-engine[api,agent] @ git+..."`, which the README never shows (#227).
4. Python / CLI door: CONFUSING.
- There is no `python -m dtk_engine`, and the README says nothing about a CLI.
- `list_keys()` returned 15 keys, and `key_schema` works.
- `dataset_overview`, `missing_values`, `duplicates` and `target_analysis` all run correctly on my CSV.
- The README snippets (`api.load`, `api.transform`, `DtkTransformer` in a sklearn pipeline, `run_key(...).show()`) all work.
- The README never shows a `source` spec (#228).
5. HTTP `dtk-api`:
- Keys, schema and run via curl: OK.
- `POST /keys/{id}/run` needs `{"params": {...}}`; a flat body gives a clear 422.
- Workspace create: CONFUSING. The natural body fails twice. The working shape is `datasets.train.x` plus `target_column`, and OpenAPI has no request schemas (#229).
6. MCP `dtk-mcp` over stdio with the `mcp` client:
- 43 tools listed, and `list_keys` works.
- `run_key` on a CSV outside `$DTK_HOME` is refused until a workspace uses the file (#230).
- `dtk-mcp doctor` works.
**Error cases (message seen by the user)**
- Wrong path (HTTP): 422 `SourceError: csv source not found: /tmp/nope.csv`. Clear; it could state the cwd.
- Unknown key: 404 `UnknownKeyError: 'nope'`. No list of valid keys (#231).
- Wrong param type (`threshold="abc"`): `threshold: Input should be a valid number, unable to parse string as a number`. Clear.
- Unknown param (`max_pct`): `max_pct: Extra inputs are not permitted`. Does not list valid params (#231).
- Target not in frame: `target 'nonexistent' not in the frame`. Does not list columns (#231).
- Source without `kind`: clear message with an example.
- Empty file: `Could not determine delimiter` (#226, comment added).
- Latin-1 file with `;` separator: parsed correctly, OK.
- `api.transform` with a missing column: raw `KeyError: "['zzz'] not found in axis"` (#233).
**Issues filed** (all in matleniz/datatoolkit-issues)
- #227 README: extras install form for non-clone users. `type:improvement`, p3-medium.
- #228 README: no "run a key on your own CSV" quickstart; no CLI stated. `type:improvement`, p3-medium.
- #229 HTTP: workspace body undocumented and OpenAPI has no request schemas. `type:improvement`, p3-medium.
- #230 MCP `run_key` refuses a CSV outside `$DTK_HOME` with no fix hint. `type:improvement`, p3-medium.
- #231 Unknown key / param / column errors do not list valid values. `type:improvement`, p4-low.
- #232 `duplicates` key returns an empty headline despite 50 duplicates. `type:bug`, p4-low.
- #233 `api.transform` missing column raises a raw pandas KeyError. `type:bug`, p4-low.
- Comments added on #224 (the README install leaves both scripts crashing, with a different missing module per script) and #226 (same message over HTTP).
- Not re-filed: #223 (malformed JSON 500) and #225 (port overflow); I did not exercise them.
**Not tested**
- The hub's HTTP-API.md was not readable under this profile; routes came from the server's `/openapi.json`.
- Docker, `agent-sdk` and the terminal packs.
- The `mcp` HTTP `/mcp` transport.
