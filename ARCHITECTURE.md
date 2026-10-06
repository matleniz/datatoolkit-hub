# Architecture — datatoolkit

Trusted-fact doc. Code in `~/datatoolkit`. If the code contradicts this file,
flag it (`propose-doc-change`), do not silently diverge.

## One implementation, three doors (built 2026-09-25, MAT-39)

Every capability lives once in `dtk_engine/ops/` and is reached through:

1. **JSON contract** (fronts) — `dtk_engine/contract.py`, see below.
2. **Notebook** — `dtk_engine/api.py`: `load(path | spec dict | spec model)`
   (suffix → source kind via `api.SUFFIX_KINDS`), `overview(df)`,
   `check(train, test, id_columns=None)`, `duplicates`, `inconsistencies`,
   `missing`, `outliers`, `select_features(df, target, ...)`, `distribution`,
   `target_analysis(df, target, ...)`, `correlations`, `chart(df, chart="histogram",
   **params)`, `preview_workspace`, `advise(df, test=None, model_family=None,
   target=None)`, `transform(df, op, **params)`, `list_transforms()`,
   `export_workspace(name, out_dir, overwrite=False, store=None, formats=None)`,
   `workspace_frames(sources)` (raw train / test from a workspace's `{datasets, label,
   merges}`, what an exported notebook starts from). DataFrame in → `Result` (displayed in Jupyter via
   `_repr_html_`) or DataFrame out. Keys and api share the same builders
   (`overview_result`, `check_result`), no duplication.
3. **sklearn** — `dtk_engine/pipeline.py`: `DtkTransformer(op, **params)`
   (pandas in/out; the op's params are the estimator params, so `clone` /
   `set_params` / grid search work; fitted `state_`, `feature_names_out_`;
   params validated at fit; `set_params(op=...)` switching op keeps only the
   current params the new op declares, MAT-85). `workspace_pipeline(name, store=None)` → unfitted
   `Pipeline` of the workspace's `both` steps only (`train` / `test`-only steps
   are one-side cleaning, skipped); empty → passthrough.

**Export** (MAT-47, `dtk_engine/workspace/export.py`):
`export_workspace(name, out_dir)` loads the raw sources (labels joined),
replays every step once (`replay_fitted`) and writes
`processed/train.parquet`, `processed/test.parquet` (if a test set; index not
kept), then `manifest.json` **last** (its presence marks a complete export).
Manifest: `generator: "dtk_engine"`, `manifest_version`, `workspace`,
`exported_at` (UTC), `versions` (dtk_engine, pandas, sklearn, pyarrow,
python), `sources` (role, part x | y, spec, resolved path, size, sha256, mtime;
a partitioned parquet dir hashed file by file), `label`, `steps` (op, target,
params, `fitted_on`, and `state` inline, or `state_file` + bytes + sha256 when
the state JSON exceeds 64 KiB — `states/step_<i>_<op>.json`), `outputs`
(path, rows, columns, sha256). Raw inputs are only read (an output path that
is a source is refused). An existing export needs `overwrite=True`, which
deletes only the files **our** manifest listed, validated first (relative,
inside `processed/` or `states/`; a manifest without the `dtk_engine` marker is
refused).
`formats` (datatoolkit-issues#156) picks the outputs: `parquet` (default), `csv`
(`processed/*.csv`), `ipynb` and `py` (`code/pipeline.*`: a replay of the steps
through the notebook door, `api.workspace_frames` + `DtkTransformer`, step and
column notes as markdown / comments; `.ipynb` is hand-built nbformat 4.5 JSON,
no dependency). The manifest adds `formats` and `outputs.{train_csv, test_csv,
notebook, script}`; `code/` is an owned dir for `overwrite`.

## Layers

```
dtk_engine  ──  contract (JSON)  ──  HTTP API (dtk-api)  ──  front (Studio)
```

| Layer | Repo · path | May import |
|---|---|---|
| Engine | `datatoolkit` · `src/dtk_engine/` | pydantic, pandas, numpy, plotly, scikit-learn, pyarrow, openpyxl, sqlalchemy, rapidfuzz, stdlib. **Never** a front. (`import dtk_engine` imports sklearn, ~1.5 s cold.) |
| HTTP API | `datatoolkit` · `src/dtk_engine/http.py` (optional extra `api`) | `dtk_engine.contract`, `dtk_engine.ui_bridge`, `dtk_engine.agent` (top level: the pure-Python `attachments` / `commands`; the rest lazily, only with the extra `agent`), FastAPI |
| UI bridge | `datatoolkit` · `src/dtk_engine/ui_bridge.py` (pure asyncio, in-memory UI context + command relay; imported only by `http.py` and `agent/`, enforced in `tests/test_layers.py`) | stdlib |
| Agent | `datatoolkit` · `src/dtk_engine/agent/` (optional extras `agent` / `agent-sdk`: MCP server, packs, chat hub, terminal, attachments; imported only by `http.py`) | `dtk_engine.contract`, `dtk_engine.ui_bridge`, `dtk_engine.errors` (`_AGENT_ALLOWED` in `tests/test_layers.py`), `mcp`, `httpx`, `claude_agent_sdk` (agent-sdk pack only), stdlib |
| Front | `datatoolkit-web` · `src/` (client: `src/api/client.ts`) | the HTTP API only (`FRONT-WEB.md`). **Never** the engine. |

Two repos since MAT-38 (2026-09-25): the engine is a standalone package
(`uv add git+https://github.com/matleniz/datatoolkit`), usable from a notebook
or scripts without any front; the front talks to it over HTTP only. The first
front (Streamlit, in-process client, MAT-38 → MAT-121) was replaced by Studio
on 2026-09-27 and its repo archived on 2026-10-01; its sections of this file
are in git history (before 2026-10-01).

## The contract (the only engine ↔ front boundary)

Defined in `dtk_engine/contract.py`. Every input and output is plain JSON
(dict / list / str / number / bool / None) — no pydantic object, numpy array or
DataFrame crosses it.

```python
def list_keys() -> list[dict]
    # [{"id": "dataset_overview", "title": "Dataset overview", "category": "analysis", "description": "...", "needs_target": false}]
    # needs_target: the key's `target` param is a required (non-nullable) column (MAT-109)
def key_schema(key_id: str) -> dict
    # JSON Schema of the key's Params (pydantic model_json_schema())
def run_key(key_id: str, params: dict) -> dict
    # Result.model_dump(mode="json"); invalid params (incl. unknown target) -> raises KeyParamsError
    # omitted source / test default to the shipped demo_data CSVs (run_key(id, {}) is not a no-op)

# workspace state (see "Workspace" below), exposed by the HTTP API
def list_workspaces() -> list[dict]          # full workspace dicts, sorted by name
def list_workspace_summaries() -> list[dict]
    # [{name, mtime, step_count, target, train:{kind,path,file,shape}, test:{...}|null}], lighter than list_workspaces
    # shape = [rows, cols] after steps, memoized content-addressed (first call may replay, later ones hit the
    # cache, MAT-200 / MAT-204); null when the role is absent or its frame cannot be loaded
def get_workspace(name: str) -> dict         # unknown -> WorkspaceNotFoundError (a KeyError)
def save_workspace(ws: dict) -> dict         # create / overwrite, returns the normalized dict; invalid shape or step params -> KeyParamsError,
    # unknown step op -> UnknownTransformError (steps checked at save, nothing written; MAT-83)
def delete_workspace(name: str) -> None
def rename_workspace(name: str, new_name: str) -> dict
def duplicate_workspace(name: str, new_name: str) -> dict
    # full copy under new_name (datasets, label, merges, variables, charts, steps); content-addressed source refs stay shared, not copied

def source_columns(spec: dict) -> list[dict]
    # columns of a source, file order: [{"name", "dtype", "numeric"}] (numeric = numeric and not bool);
    # the options of a column-selector param; invalid spec -> KeyParamsError, unreadable source -> SourceError

# transforms (steps of a workspace)
def list_transforms() -> list[dict]          # [{"op", "title", "description", "needs_target"}]
def transform_schema(op: str) -> dict        # JSON Schema of the op's params; unknown -> UnknownTransformError (a KeyError)
def preview_workspace(ws: dict, role: str, head_rows: int = 5) -> dict
    # unsaved workspace dict, validated like save_workspace, replayed in memory (no store write):
    # {"shape": [rows, cols], "columns": [...], "head": records}; bad role / invalid ws -> KeyParamsError

# export (see "Export" above)
def export_workspace(name: str, out_dir: str, overwrite: bool = False, formats: list[str] | None = None) -> dict
    # writes the chosen formats (parquet default, csv, ipynb, py) + manifest.json, returns the manifest; unknown -> WorkspaceNotFoundError;
    # existing export without overwrite / output == raw input -> KeyParamsError; source or step failing on the data -> SourceError;
    # unknown step op -> UnknownTransformError, invalid step params -> KeyParamsError
```

Errors (`dtk_engine/errors.py`): `UnknownKeyError` (unknown key id),
`UnknownTransformError` (unknown transform op),
`KeyParamsError` (params fail validation — incl. a misspelled param or an
unknown source `kind`, or invalid workspace step params), `SourceError` (a
source cannot be loaded — missing / empty / unreadable file — or a workspace
step fails on the data; raised unchanged by `run_key`). An unknown step op is
`UnknownTransformError` everywhere (save, preview, replay, export), never
`SourceError` (MAT-84).

**Studio additions (built 2026-09-27, MAT-126; datatoolkit#30–33)** — for the
web front (`FRONT-WEB.md`, repo `datatoolkit-web`), backed by
`dtk_engine/workspace/inspect.py`:

```python
def workspace_rows(ws, role, version=None, offset=0, limit=500, columns=None, filter=None, sort=None) -> dict
    # {columns[{name,dtype,kind,semantic}], rows[... + _rid], total, total_unfiltered, version}; version = steps replayed (time travel);
    # filter = the filter_rows params shape {conditions, combine}; sort = [{column, desc}] (stable, NaN last);
    # both view-only, applied after the steps and before paging; total = rows after filter, total_unfiltered = before
    # (datatoolkit-issues#87)
    # _rid = row position in the raw (post-label, post-merge) frame, kept through row-dropping steps
    # columns: non-empty list restricts column meta + row cells to those names, in order; unknown -> KeyParamsError (MAT-152)
def column_profiles(ws, role, version=None, columns=None) -> dict   # {columns[profile], version}
    # columns: non-empty list profiles only those names, in order; unknown -> KeyParamsError; None/empty = all (MAT-152)
    # profile: kind, count, missing, sentinel_candidates, distinct, histogram | top_values,
    # iqr_bounds, outliers, variants, looks_like_dates, numbers_as_text (',' or '.'), skewed
def preview_step(ws, step, role) -> dict
    # ws + step replayed in memory: shape, columns, added/removed_columns, removed_rids,
    # changed[{_rid,column,before,after}] (capped) + changed_total, state (fitted), fitted_on
def align_report(ws) -> dict
    # {columns[{train,test,status,numbers_as_text,train_mean,test_mean,similar,
    #   only_in_test,pct_test_rows_unseen,near_match_hint,near_matches,blocking}]}
    # status: match | type_mismatch | value_mismatch | missing_in_test | extra_in_test | label
    # value_mismatch = a categorical column has test values unseen in train (MAT-155);
    #   blocking: true if near_matches (likely spelling variants) or pct_test_rows_unseen > 50,
    #   else informational (rare new categories one-hot handle_unknown absorbs, MAT-179)
```

kinds: `number | binary | bool | text | date | identifier` (semantic_type + an
`id` / `*_id` name heuristic). Workspaces gain `merges` (left join
many-to-one of an extra file on a key, `apply_to` train | both, refuses
duplicated keys / clashes / row loss, applied after the label join and before
steps, `ops/join.py::merge_table`) and `variables` (`[{name, stat, column}]`,
stored for the front; formula steps carry their own snapshot) and `charts`
(`[{name, params}]`: chart-key params minus `source`, unique names, saved by
the Studio chart builder for reload — MAT-185). The HTTP API
`dtk_engine/http.py` (extra `api`, `dtk-api --port 8765`) exposes the whole
contract 1:1 — routes and error mapping in `FRONT-WEB.md`. Read-only compute
routes (`keys/{id}/run`, `source/columns`, workspace preview / rows /
profiles / preview-step / align) run the engine through `run_in_threadpool`,
so a slow key does not stall previews; store writes stay on the event loop
(MAT-212).

## Column-selector params (MAT-95)

A key param built with `params.columns_field(...)` (`list[str]`, default `[]`
= every eligible column, capped per key; `nullable=True` keeps a "null = auto"
default) or `params.column_field(...)` (`str`, e.g. a target) carries in its
JSON Schema:

| hint | values | meaning |
|---|---|---|
| `x-dtk-widget` | `columns` / `column` | multiselect / single choice |
| `x-dtk-source` | name of a sibling param, or `"step"` | the `SourceSpec` param whose columns are the options; `"step"` (transform ops, MAT-119) = the frame the step applies to (workspace `dataset` source of the step's role, state before the step) |
| `x-dtk-dtype` | `any` / `numeric` | offer every column / only numeric ones |
| `x-dtk-when` | `{sibling param: value or list of values}` | any param: it only applies (the front shows it) when every listed sibling has that value, e.g. `impute.expr` → `{"strategy": "formula"}`, `impute.by` → `{"strategy": ["group_mean", "group_prev", "group_interp"]}` |
| `x-dtk-semantic` | a `semantic_type`, e.g. `group_id` | single-column param a front may prefill with the column of that semantic type (`workspace_rows` column meta `semantic`); `impute.by`, `ffill.by` |

The front calls `source_columns(<value of the x-dtk-source param>)` (a
workspace `dataset` source works too) and falls back to a free-text field if
that fails. Used by every key with a target or column-list param
(`column_distribution`, `target_analysis`, `correlations`, `feature_selection`,
`missing_values`, `preprocessing_advisor`, `train_test_check.id_columns`, …).

## Result (engine output)

`dtk_engine/result.py`, pydantic model:

| Field | Type | Meaning |
|---|---|---|
| `headline` | `str` | one plain English sentence summarising the finding (e.g. "2 columns have missing values; 1 above 30 %"); `""` when nothing to say or for `chart` (MAT-244) |
| `metrics` | `dict[str, float \| int \| str]` | headline numbers |
| `tables` | `list[Table]` — `{"title": str, "records": list[dict], "group": str \| null, "kind": "steps" \| null}` | tabular output; `kind: "steps"` = each record is a workspace step (`op`, `target`, `params`) — advisor recommendations, feature_selection / correlations suggested steps |
| `figures` | `list[Figure]` — `{"title": str, "plotly": dict, "group": str \| null, "main": bool}` | Plotly figure JSON (`json.loads(fig.to_json())`); at most one `main: true` = the figure a front opens by default (none → the first) (MAT-244) |
| `text` | `str` | markdown commentary, may be empty |

Helpers: `Result.add_figure(title, fig, group=None, main=False)` (a later
`main=True` clears the previous one; figure construction and serialization run
under the process-wide `plotly_lock`, since Plotly is not thread-safe and the
HTTP API runs keys in a threadpool — MAT-243),
`Result.add_table(title, df, group=None, kind=None)` (convert to JSON-safe records).
`group` is optional: a front renders one tab per group in first-appearance
order, ungrouped items in a leading "Overview" tab; nothing grouped → flat
layout. In a notebook a `Result` displays itself
(`_repr_html_`: metrics table, first `HTML_TABLE_ROWS` = 10 rows of each
table, figures with plotly.js loaded once from the CDN, escaped text), so the
idiom is `api.overview(df)`; `Result(**run_key(id, {})).show()` still works.

Keys use absolute imports (`from dtk_engine.registry import key`); no module
uses a relative parent import (not a lint rule; check with
`uv run ruff check --select TID252 .`).

## A key

```python
# dtk_engine/keys/<id>.py
class Params(KeyParams): ...          # strict (extra="forbid"), defaults = sensible demo
@key(id="<id>", title="...", category="...", description="...")
def run(params: Params) -> Result: ...  # load(source) -> ops -> Result
```

`KeyParams` lives in `dtk_engine/params.py` (also exported from `dtk_engine`).
`registry.py` holds the `@key` decorator and the registry; `keys/__init__.py`
imports every key module so registration happens on `import dtk_engine`.
Recipe: `HOWTO/add-a-key.md`.

## Front (generic forms and results)

A front renders the contract generically: forms from `key_schema` /
`transform_schema` (JSON Schema → widgets, `x-dtk-widget` column selectors),
results from `Result` (metrics, tables, figures, text). The front is Studio —
screens, routes and tests in `FRONT-WEB.md`; it has no key catalog (its tool
rail and Suggestions call a fixed list of keys). Switching front = implement
`HOWTO/add-a-front.md`, touch nothing in the engine. Deployment: nginx + engine
containers (`HOWTO/run-with-docker.md`), or `dtk-studio` (datatoolkit-web
`launcher/`), which mounts the engine's `create_app()` and Studio's static build
in one ASGI app (`HOWTO/share-and-run.md`); either way Studio talks to the
engine over HTTP only.

## Engine internals: sources → ops → keys

Principle: **one generalized input → one output.** Behind the unchanged contract
the engine splits into three internal layers:

```
dtk_engine/
  sources/   SourceSpec (JSON) ──load()──▶ pd.DataFrame        generalized input
  ops/       pure functions: DataFrame(s) ─▶ DataFrame / dict   reused by keys and pipelines
  ops/transforms/{cleaning,align,impute,encode,scale,features,selection,formula}.py   fit/apply ops (steps)
  ops/advisor/, ops/compare/   packages (stage / concern modules); shared helpers in ops/_util.py
  ops/groups.py   within-entity fills (group_mean / group_prev / group_interp), shared by formula and impute
  keys/      thin: Params(sources…) → load → ops → Result      one output
```

Import order inside `dtk_engine` (a module only imports the layers below it):
`http` → `contract` → `api` | `pipeline` (siblings) → `workspace` → `keys` → `ops`
→ `sources`. `contract` (JSON door) and `api` (notebook door) both build on
`workspace` / `keys` / `transform_registry`; neither imports the other (datatoolkit-issues#4).
Enforced by `tests/test_layers.py` (dependency-free `ast` test, part of the
gate; datatoolkit-issues#3). Ruff also caps complexity (`C90` max 10,
`PLR0911/0912/0915`) and selects `I`, `SIM`, `PERF`, `B`, `RUF100`.

- `sources/` never knows about keys; analysis `ops/` never know about
  `Result` or pydantic (testable alone); transform ops declare pydantic params
  (`TransformParams`) so fronts build their forms; `keys/` only glue.
- **Canonical frame = pandas** for now: every reader returns a `pd.DataFrame`.
  A future polars/duckdb reader converts to pandas at the boundary. Making ops
  backend-agnostic is a separate, later decision.
- Params are strict: every key's `Params` derives from a `KeyParams` base with
  `extra="forbid"` (unknown/misspelled params raise `KeyParamsError`).

### Caches (MAT-263, 2026-10-01)

All caching lives in `dtk_engine/cache.py`: a bounded thread-safe `LRU`, a
content-addressed `digest` (JSON of the request + `os.stat` of every file
read, or a value hash of the frame), `memoize` and `memo_frame`. It covers raw
sources, replayed workspace frames and shapes, seeded model fits (MAT-210),
per-column facts (`semantic_type`, `numeric_text_format`, column profiles) and
`advise`. There are no invalidation hooks: a key is the content, so a changed
step, dataset or file yields a new key and old entries age out of the LRU.
Effect on the Parkinson workspace (warm): `column_profiles` 1.6 s → 0.04 s,
`preprocessing_advisor` 1.05 s → 0.03 s, `workspace_rows` 0.37 s → 0.04 s.

## Inputs: SourceSpec

A pydantic union discriminated on `kind`; a key declares e.g.
`source: SourceSpec`. Readers register with `@reader(kind)` (same pattern as
`registry.py`) and `load(spec)` dispatches. A missing/unreadable file raises a
`SourceError`.

Files: `sources/spec.py` (`CsvSource`, `ParquetSource`, `ExcelSource`,
`JsonSource`, `SqlSource`, `DatasetSource`, the `SourceSpec` union
— **add every new reader's spec to this union, and file readers also to
`FileSourceSpec`**, the file-only union a workspace uses for X / y so it cannot
reference a `dataset` source), `sources/registry.py` (`@reader`, `load`),
`sources/csv_pandas.py`, `parquet.py`, `excel.py`, `json_reader.py`, `sql.py`;
the `dataset` reader is `workspace/dataset.py::read_dataset`. Spec models are strict (`extra="forbid"`). `api.load(path)` maps
`.csv` / `.tsv` / `.parquet` / `.xlsx` / `.json` / `.jsonl` / `.ndjson` through
`api.SUFFIX_KINDS`; legacy `.xls` maps to `excel` only to raise a clear
`SourceError` (only `.xlsx` is supported, no extra dependency; datatoolkit-issues#18).

| `kind` | Reader | Status |
|---|---|---|
| `csv` | pandas `read_csv`: `path`, `sep` (`"auto"` → `csv.Sniffer` on the first 64 KB among `,` `;` tab `\|`, then C engine; fallback `sep=None, engine="python"`; if the Sniffer fails, a single candidate on the header line wins), `encoding` (`"auto"` default: BOM, else utf-8, else cp1252, latin-1 last; explicit values strict), `decimal` (`"auto"` default: `,` for non-comma files holding more `1,5` than `1.5`), `header` (null = no header), `na_values`, `dtype`, `parse_dates`, `usecols`, `on_bad_lines` (`error` default: strict pre-scan → `SourceError` naming the line on unclosed quote / text after a closing quote / too many fields; `warn` / `skip` = pandas recovery), `keep_leading_zeros` (default true: digits-only columns with leading zeros stay strings). `skiprows` (`int | "auto"`, default 0: physical junk lines dropped before parsing, `header` counts after them; `"auto"` detects them — first line whose field count matches the body — and also fixes a delimiter mis-sniffed because of them), `mixed_sep` (`"error"` default: lines using another delimiter than `sep` raise `MixedSeparatorError`, a `SourceError` with 1-based `.lines`; `"normalize"` rewrites them when safe — no quote, no `sep` on the line — else still an error; `"ignore"` = previous behaviour) (datatoolkit-issues#59). The file is read once as bytes and parsed from memory (MAT-61…69). A Windows `path` (`C:\…`, `C:/…`) that does not exist is mapped to `/mnt/<drive>/…` on Linux/WSL. A single-column file (no candidate separator in the header) reads with `,` | done |
| `dataset` | current state of a workspace dataset: `workspace`, `role` (`train` \| `test`), `labeled` (default true) → load X, join y (or keep / drop `target_column` per `labeled`), replay the steps for the role | done |
| `parquet` | pyarrow via pandas: `path` (file or partitioned dir), `columns`, `filters` (`[column, op, value]`, ANDed) | done (MAT-40) |
| `excel` | openpyxl: `path`, `sheet` (name or index), `header` (0-based; `file_inspect` suggests one per sheet), `usecols` | done (MAT-40) |
| `json` | `path`, `lines` (jsonl), `encoding` (`utf-8-sig` default, BOM-safe), `record_path` (dotted; `file_inspect` suggests one); nested objects flattened `a_b`, lists kept as-is | done (MAT-40) |
| `sql` | SQLAlchemy: `url_env` = NAME of an env var holding the URL (never the URL: it would land in workspaces / logs), `query` run on the server; errors never echo the URL. Not a file source (not usable as workspace X / y) | done (MAT-40) |
| `csv_robust` | not a kind: bad lines, junk header lines and mixed separators are `csv` options (`on_bad_lines`, `skiprows`, `mixed_sep`) | done (#59) |
| (upload) | not a kind: `PUT /api/uploads/{filename}` stores the bytes as `$DTK_UPLOAD_DIR` (default `$DTK_HOME/uploads`)`/<content hash>/<name>`; the returned path is read with its file kind (`csv`, …) | done (MAT-102) |

Step 1 input = a local path. Default paths point to the demo CSVs shipped in
`dtk_engine/demo_data/` (`TRAIN_CSV`, `TEST_CSV`) so `run_key(id, {})` stays
runnable. They are synthetic Titanic-like with deliberate train/test
inconsistencies (`Survived` only in train, `Embarked="Q"` only in test, `Age`
numeric in train / text in test, one duplicated train row).


## Workspace: loaded datasets + transform log (built 2026-09-25)

Capability families and backlog: `CAPABILITIES.md`.

A **workspace** is the engine-side state of one project: which files make train
and test, how the labels join, and the ordered log of transforms. It lives as
JSON in `~/.datatoolkit/workspaces/<name>.json` behind a `WorkspaceStore`
interface (JSON file today; materialized tables, e.g. parquet, later and only
with approval). Any front (Studio, notebook, scripts) sees the same state.

```
workspace "parkinson"
  datasets:
    train : X = SourceSpec   y = SourceSpec | null   (or target column already in X)
    test  : X = SourceSpec   y = null
  label   : mode "order" (y has one column, same row count) | "key" (common column)
  steps   : [ {id, op, target: train|test|both, params, note?}, ... ]   ← full trace
  notes   : {workspace: text?, columns: {origin name: text}}   # free text, max 4,000 chars
  charts  : [ {name, params}, ... ]   # saved chart-key params (minus source); unique names
```

- **Step ids and notes** (datatoolkit-issues#153, #152): a step `id` (`s` +
  opaque text) names its pipeline slot, kept on replace; ids missing in an
  older workspace read `s<position>`. Notes are not data: `id` and `note` stay
  out of the cache key and the data identity. A column note is keyed by the
  column's origin name; `contract.column_notes` / `POST
  /api/workspace/column-notes` resolve names at a version through `rename` /
  `rename_columns_bulk`.
- **Current state = sources + steps replayed in order.** Undo = drop the last
  step and replay. The step log *is* the future reproducible pipeline (add input
  hashes + versions → run manifest).
- **Keys stay unchanged**: a new SourceSpec `kind="dataset"`
  (`workspace`, `role: train|test`, `labeled`) loads the current state, so every
  analysis key runs on the transformed data.
- **Label join** (X + y): `order` (y has a single value column, same row count)
  or `key` (join column). It refuses to lose rows. A preview key showing X / Y
  columns and join candidates before joining is planned (see `CAPABILITIES.md`),
  not built.
- **Transform ops** are workspace steps applied to `train`, `test` or `both`,
  following the fit/apply protocol below (stat-based ops fit on train and apply
  to test). Catalog: `TRANSFORMS.md`.
- **Front**: Studio's header shows the active workspace name and, on the
  Workbench, a Train / Test toggle; analysis keys get a `{kind: "dataset"}`
  source for the workspace (`FRONT-WEB.md`, Refresh identity).

Implementation:
- Package `dtk_engine/workspace/`: `models.py` (strict pydantic `Workspace`:
  `datasets.train` required, `datasets.test` optional; per dataset `y` and
  `target_column` are mutually exclusive; `label.key` required for mode `key`),
  `store.py` (`WorkspaceStore` protocol, `JsonWorkspaceStore`: root
  `$DTK_HOME/workspaces`, default `~/.datatoolkit/workspaces`, or a `root` arg;
  atomic writes; name pattern `^[A-Za-z0-9][A-Za-z0-9_.-]{0,63}$`; save / delete
  / rename / duplicate take an exclusive flock on `root/.lock` so concurrent
  `PUT`s cannot lose an update, MAT-171), `replay.py`.
- **Transform protocol** (`dtk_engine/transform_registry.py`, MAT-39):
  `@transform(op, params_model=<TransformParams subclass>, fit=<optional>,
  title=?, description=?)` on `apply(df, params, state) -> df`;
  `fit(df, params) -> state` must return a JSON-safe dict (checked; stateless
  ops get `{}`); title defaults to the op name, description to the first
  docstring line. Replay (`workspace/replay.py`): `both` → fit on train as of
  that step, apply to train and test; `train` / `test` → fit and apply on that
  role. Unknown op → `UnknownTransformError`, invalid params → `KeyParamsError`
  (same check as `save_workspace`, `replay.validate_steps`), both before any
  work; an op failing on the data → `SourceError` naming the step (MAT-84).
  Ops live in `ops/transforms/<family>.py` (all imported by its `__init__`);
  `drop_columns` is the reference op. Recipe: `HOWTO/add-a-transform.md`.
  Supervised ops declare `needs_target=True` (their params must have a
  `target` field): `DtkTransformer.fit(X, y)` joins `y` to X under that name
  for fit only (MAT-56); unsupervised ops ignore `y`.
  `replay.replay_fitted(steps, train, test) -> (train, test, fitted)` is the
  single-pass variant that keeps each step's fitted state (`fitted_on` train |
  test), used by the export; `replay` and `replay_fitted` share one loop
  (`_run`, MAT-79), as does `replay.replay_step(step, index, train, test) ->
  (train, test, fitted)`, which fits and applies one more step on frames
  already replayed through the first `index` steps: `preview_step` uses it on
  the cached post-log frames, so a preview fits only the pending step
  (MAT-212). The `dataset` source lives in `workspace/dataset.py`
  (MAT-263): `raw_workspace_frame` (labels + merges, no steps),
  `workspace_frame` (steps replayed, cached), `parse_workspace` (workspace dict
  → model + step validation, used by every contract call taking an unsaved
  workspace) and `preview` (behind `contract.preview_workspace` /
  `api.preview_workspace`). `list_transforms` serves
  `transform_registry.transform_catalog()`.
- **Label join** `ops/join.py::join_labels(x, y, mode, key)`: `order` = y has one
  value column plus an optional index-like column (named index / idx /
  `Unnamed: 0`, or integers 0..n-1 / 1..n) or the `key` column; if that column
  also exists in X it must be equal row by row (real data: `Index` checked).
  `key` = one-to-one, no NaN keys, no unmatched row on either side. Both refuse
  row loss; the error lists the counts. X's row order and index are kept.
- `tests/test_real_data.py` runs on Matteo's real CSVs, skipped when absent.
