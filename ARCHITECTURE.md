# Architecture — datatoolkit

Trusted-fact doc. Code in `~/datatoolkit`. If the code contradicts this file,
flag it (`propose-doc-change`), do not silently diverge.

## One implementation, three doors (built 2026-09-25, MAT-39)

Every capability lives once in `dtk_engine/ops/` and is reached through:

1. **JSON contract** (fronts) — `dtk_engine/contract.py`, see below.
2. **Notebook** — `dtk_engine/api.py`: `load(path | spec dict | spec model)`
   (suffix → source kind via `api.SUFFIX_KINDS`), `overview(df)`,
   `check(train, test, id_columns=None)`, `duplicates`, `inconsistencies`,
   `missing`, `outliers`, `advise(df, test=None, model_family=None,
   target=None)`, `transform(df, op, **params)`, `list_transforms()`,
   `export_workspace(name, out_dir, overwrite=False, store=None)`. DataFrame in → `Result` (displayed in Jupyter via
   `_repr_html_`) or DataFrame out. Keys and api share the same builders
   (`overview_result`, `check_result`), no duplication.
3. **sklearn** — `dtk_engine/pipeline.py`: `DtkTransformer(op, **params)`
   (pandas in/out; the op's params are the estimator params, so `clone` /
   `set_params` / grid search work; fitted `state_`, `feature_names_out_`;
   params validated at fit). `workspace_pipeline(name, store=None)` → unfitted
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

## Layers

```
dtk_engine  ──  contract (JSON)  ──  EngineClient  ──  front (dtk_streamlit today, web tomorrow)
```

| Layer | Repo · path | May import |
|---|---|---|
| Engine | `datatoolkit` · `src/dtk_engine/` | pydantic, pandas, plotly, scikit-learn, pyarrow, openpyxl, sqlalchemy, stdlib. **Never** a front. (`import dtk_engine` imports sklearn, ~1.5 s cold.) |
| Client | `datatoolkit-streamlit` · `src/dtk_streamlit/client.py` | `dtk_engine.contract` only |
| Front | `datatoolkit-streamlit` · `src/dtk_streamlit/` (all other modules) | `dtk_streamlit.client`, streamlit, plotly. **Never** `dtk_engine`. |

Two repos since MAT-38 (2026-09-25): the engine is a standalone package
(`uv add git+https://github.com/matleniz/datatoolkit`, no streamlit); the front
depends on it via git (`dtk-engine @ git+…/datatoolkit`, locked in its
`uv.lock` — re-run `uv lock` there to pick up a new engine commit).

Enforced mechanically in the front repo: ruff `TID251` bans `dtk_engine`
except `client.py`; `tests/test_front_isolation.py` asserts the same. Both run
in its `fleet gate`.

## The contract (the only engine ↔ front boundary)

Defined in `dtk_engine/contract.py`. Every input and output is plain JSON
(dict / list / str / number / bool / None) — no pydantic object, numpy array or
DataFrame crosses it.

```python
def list_keys() -> list[dict]
    # [{"id": "dataset_overview", "title": "Dataset overview", "category": "analysis", "description": "..."}]
def key_schema(key_id: str) -> dict
    # JSON Schema of the key's Params (pydantic model_json_schema())
def run_key(key_id: str, params: dict) -> dict
    # Result.model_dump(mode="json"); invalid params -> raises KeyParamsError

# workspace state (see "Workspace" below), mirrored in EngineClient / LocalClient
def list_workspaces() -> list[dict]          # full workspace dicts, sorted by name
def get_workspace(name: str) -> dict         # unknown -> WorkspaceNotFoundError (a KeyError)
def save_workspace(ws: dict) -> dict         # create / overwrite, returns the normalized dict; invalid -> KeyParamsError
def delete_workspace(name: str) -> None

# transforms (steps of a workspace)
def list_transforms() -> list[dict]          # [{"op", "title", "description"}]
def transform_schema(op: str) -> dict        # JSON Schema of the op's params; unknown -> UnknownTransformError (a KeyError)

# export (see "Export" above)
def export_workspace(name: str, out_dir: str, overwrite: bool = False) -> dict
    # writes processed parquet + manifest.json, returns the manifest; unknown -> WorkspaceNotFoundError;
    # existing export without overwrite / output == raw input -> KeyParamsError; source or step failure -> SourceError
```

Errors (`dtk_engine/errors.py`): `UnknownKeyError` (unknown key id),
`UnknownTransformError` (unknown transform op),
`KeyParamsError` (params fail validation — incl. a misspelled param or an
unknown source `kind`), `SourceError` (a source cannot be loaded: missing /
empty / unreadable file; raised unchanged by `run_key`). The Streamlit front
shows any engine error via `st.error`.

An HTTP API later exposes exactly these (`GET /keys`,
`GET /keys/{id}/schema`, `POST /keys/{id}/run`, `GET /workspaces`,
`GET|PUT|DELETE /workspaces/{name}`) with zero per-key code.

## Result (engine output)

`dtk_engine/result.py`, pydantic model:

| Field | Type | Meaning |
|---|---|---|
| `metrics` | `dict[str, float \| int \| str]` | headline numbers |
| `tables` | `list[Table]` — `{"title": str, "records": list[dict], "group": str \| null}` | tabular output |
| `figures` | `list[Figure]` — `{"title": str, "plotly": dict, "group": str \| null}` | Plotly figure JSON (`json.loads(fig.to_json())`) |
| `text` | `str` | markdown commentary, may be empty |

Helpers: `Result.add_figure(title, fig, group=None)`,
`Result.add_table(title, df, group=None)` (convert to JSON-safe records).
`group` is optional: a front renders one tab per group in first-appearance
order, ungrouped items in a leading "Overview" tab; nothing grouped → flat
layout. Streamlit also gives any table with a `column` field a multiselect
filter on it (generic; pure helpers `group_items`, `filter_options`,
`filter_records` in `render.py`). In a notebook a `Result` displays itself
(`_repr_html_`: metrics table, first `HTML_TABLE_ROWS` = 10 rows of each
table, figures with plotly.js loaded once from the CDN, escaped text), so the
idiom is `api.overview(df)`; `Result(**run_key(id, {})).show()` still works.

Keys use absolute imports (`from dtk_engine.registry import key`): ruff TID252
forbids relative parent imports.

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

## Front (generic, zero per-key code)

`dtk_streamlit/app.py`: sidebar radio over `client.list_keys()`, labelled
`category / title`, sorted by category → `render.form_from_schema(schema)` builds widgets from JSON Schema
(integer, number, string, boolean, enum; objects/unions → recursive sub-forms, see
"Inputs: SourceSpec"; unknown types → JSON text input) →
`client.run_key` → `render.result(result_dict)` shows metrics, tables
(`st.dataframe`), figures (`st.plotly_chart(plotly.io.from_json(...))`), text.

**Transforms panel** (MAT-46, `app.py::transforms_panel` + pure
`steps.py`): under the workspace bar, lists `client.list_transforms()`, builds
the op form from `client.transform_schema(op)` (same `schema.py` machinery),
target train / test / both, "Add step" / "Undo last step" via `save_workspace`,
step log. "Preview step" shows shape + head before / after through the
`dataset` source; today it saves and deletes a scratch workspace
`<name>.preview` (store side effect — MAT-55 moves the preview into the
engine, no write).

`EngineClient` is a `typing.Protocol` mirroring the contract (keys,
workspaces, `list_transforms`, `transform_schema`).
`LocalClient` calls `dtk_engine.contract` in-process. `HttpClient` = future.
Switching front = implement `HOWTO/add-a-front.md`, touch nothing in the engine.

## Engine internals: sources → ops → keys

Principle: **one generalized input → one output.** Behind the unchanged contract
the engine splits into three internal layers:

```
dtk_engine/
  sources/   SourceSpec (JSON) ──load()──▶ pd.DataFrame        generalized input
  ops/       pure functions: DataFrame(s) ─▶ DataFrame / dict   reused by keys and pipelines
  ops/transforms/{cleaning,impute,encode,scale,features}.py   fit/apply ops (steps)
  keys/      thin: Params(sources…) → load → ops → Result      one output
```

- `sources/` never knows about keys; analysis `ops/` never know about
  `Result` or pydantic (testable alone); transform ops declare pydantic params
  (`TransformParams`) so fronts build their forms; `keys/` only glue.
- **Canonical frame = pandas** for now: every reader returns a `pd.DataFrame`.
  A future polars/duckdb reader converts to pandas at the boundary. Making ops
  backend-agnostic is a separate, later decision.
- Params are strict: every key's `Params` derives from a `KeyParams` base with
  `extra="forbid"` (unknown/misspelled params raise `KeyParamsError`).

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
`sources/csv_pandas.py`, `parquet.py`, `excel.py`, `json_reader.py`, `sql.py`,
`dataset.py`. Spec models are strict (`extra="forbid"`). `api.load(path)` maps
`.csv` / `.tsv` / `.parquet` / `.xlsx` / `.json` / `.jsonl` through
`api.SUFFIX_KINDS`.

| `kind` | Reader | Status |
|---|---|---|
| `csv` | pandas `read_csv`: `path`, `sep` (`"auto"` → `csv.Sniffer` on the first 64 KB among `,` `;` tab `\|`, then C engine; fallback `sep=None, engine="python"`), `encoding`, `decimal`, `header` (null = no header), `na_values`, `dtype`, `parse_dates`, `usecols`. A Windows `path` (`C:\…`, `C:/…`) that does not exist is mapped to `/mnt/<drive>/…` on Linux/WSL. A single-column file (no candidate separator in the header) reads with `,` | done |
| `dataset` | current state of a workspace dataset: `workspace`, `role` (`train` \| `test`), `labeled` (default true) → load X, join y (or keep / drop `target_column` per `labeled`), replay the steps for the role | done |
| `parquet` | pyarrow via pandas: `path` (file or partitioned dir), `columns`, `filters` (`[column, op, value]`, ANDed) | done (MAT-40) |
| `excel` | openpyxl: `path`, `sheet` (name or index), `header`, `usecols` | done (MAT-40) |
| `json` | `path`, `lines` (jsonl), `record_path` (dotted); nested objects flattened `a_b`, lists kept as-is | done (MAT-40) |
| `sql` | SQLAlchemy: `url_env` = NAME of an env var holding the URL (never the URL: it would land in workspaces / logs), `query` run on the server; errors never echo the URL. Not a file source (not usable as workspace X / y) | done (MAT-40) |
| `csv_robust` | malformed CSVs (bad lines, mixed separators, junk headers) | later |
| `upload` | file dropped by the front into a staging dir | later |

Step 1 input = a local path. Default paths point to the demo CSVs shipped in
`dtk_engine/demo_data/` (`TRAIN_CSV`, `TEST_CSV`) so `run_key(id, {})` stays
runnable. They are synthetic Titanic-like with deliberate train/test
inconsistencies (`Survived` only in train, `Embarked="Q"` only in test, `Age`
numeric in train / text in test, one duplicated train row).

The Streamlit front renders object params recursively (sub-form per object,
`const` fields fixed, selectbox on `kind` when a union has several members,
parent defaults flow into the sub-form) — still zero per-key code. The logic is
the pure module `dtk_streamlit/schema.py` (`build_params(schema, widgets)`, a
`Widgets` protocol injected by `render.py`), unit-tested without Streamlit in
`tests/front/test_schema_form.py` (front repo).

## Workspace: loaded datasets + transform log (built 2026-09-25)

Capability families and backlog: `CAPABILITIES.md`.

A **workspace** is the engine-side state of one project: which files make train
and test, how the labels join, and the ordered log of transforms. It lives as
JSON in `~/.datatoolkit/workspaces/<name>.json` behind a `WorkspaceStore`
interface (JSON file today; materialized tables, e.g. parquet, later and only
with approval). Any front (Streamlit, notebook, future web) sees the same state.

```
workspace "parkinson"
  datasets:
    train : X = SourceSpec   y = SourceSpec | null   (or target column already in X)
    test  : X = SourceSpec   y = null
  label   : mode "order" (y has one column, same row count) | "key" (common column)
  steps   : [ {op, target: train|test|both, params}, ... ]   ← full trace
```

- **Current state = sources + steps replayed in order.** Undo = drop the last
  step and replay. The step log *is* the future reproducible pipeline (add input
  hashes + versions → run manifest).
- **Keys stay unchanged**: a new SourceSpec `kind="dataset"`
  (`workspace`, `role: train|test`, `labeled`) loads the current state, so every
  analysis key runs on the transformed data.
- **Label join** (X + y): `order` (y has a single value column, same row count)
  or `key` (join column). It refuses to lose rows. A preview key
  (`label_join_preview`) shows X / Y columns and join candidates before joining.
- **Transform ops** are pure functions in `ops/`: `(df, params) -> df`, applied
  to `train`, `test` or `both`. Stat-based ops can fit on train and apply to
  test (e.g. realign a shifted test column with train statistics); the exact
  per-op options are decided with Matteo when each op is built.
- **Front**: a top bar shows the active workspace (train / test / y, number of
  steps); key forms are pre-filled with the workspace datasets.

Implementation:
- Package `dtk_engine/workspace/`: `models.py` (strict pydantic `Workspace`:
  `datasets.train` required, `datasets.test` optional; per dataset `y` and
  `target_column` are mutually exclusive; `label.key` required for mode `key`),
  `store.py` (`WorkspaceStore` protocol, `JsonWorkspaceStore`: root
  `$DTK_HOME/workspaces`, default `~/.datatoolkit/workspaces`, or a `root` arg;
  atomic writes; name pattern `^[A-Za-z0-9][A-Za-z0-9_.-]{0,63}$`), `replay.py`.
- **Transform protocol** (`dtk_engine/transform_registry.py`, MAT-39):
  `@transform(op, params_model=<TransformParams subclass>, fit=<optional>,
  title=?, description=?)` on `apply(df, params, state) -> df`;
  `fit(df, params) -> state` must return a JSON-safe dict (checked; stateless
  ops get `{}`); title defaults to the op name, description to the first
  docstring line. Replay (`workspace/replay.py`): `both` → fit on train as of
  that step, apply to train and test; `train` / `test` → fit and apply on that
  role. Unknown op → `SourceError`, invalid params → `KeyParamsError`, both
  before any work; an op failing on the data → `SourceError` naming the step.
  Ops live in `ops/transforms/<family>.py` (all imported by its `__init__`);
  `drop_columns` is the reference op. Recipe: `HOWTO/add-a-transform.md`.
  `replay.replay_fitted(steps, train, test) -> (train, test, fitted)` is the
  single-pass variant that keeps each step's fitted state (`fitted_on` train |
  test), used by the export; `sources/dataset.py::labeled_frame` is public.
- **Label join** `ops/join.py::join_labels(x, y, mode, key)`: `order` = y has one
  value column plus an optional index-like column (named index / idx /
  `Unnamed: 0`, or integers 0..n-1 / 1..n) or the `key` column; if that column
  also exists in X it must be equal row by row (real data: `Index` checked).
  `key` = one-to-one, no NaN keys, no unmatched row on either side. Both refuse
  row loss; the error lists the counts. X's row order and index are kept.
- Front: forms prefilled by the pure `schema.workspace_defaults` (first source
  param → train, a param named `test` → test, others keep their default); the
  widget key prefix includes the active workspace so forms reset on switch.
  Headless `AppTest` smoke tests in `tests/front/test_app.py` (front repo).
- `tests/test_real_data.py` runs on Matteo's real CSVs, skipped when absent.
