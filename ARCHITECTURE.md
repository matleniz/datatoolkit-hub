# Architecture — datatoolkit

Trusted-fact doc. Code in `~/datatoolkit`. If the code contradicts this file,
flag it (`propose-doc-change`), do not silently diverge.

## Planned (validated 2026-09-25, not built yet)

Status: **planned** — the sections below describe what is built today; this
block becomes the reference as MAT-39 merges (detail travels with the
merge).

- **Two repos** — built (MAT-38), see "Layers".
- **One implementation, three doors.** Every capability lives once in
  `dtk_engine/ops/` and is reached through:
  1. the JSON contract (fronts): `run_key`, workspaces, plus
     `list_transforms()` / `transform_schema(op)`;
  2. the notebook facade `dtk_engine.api`: DataFrame in → `Result` (rendered by
     `_repr_html_`) or DataFrame out — `load`, `overview`, `check`, `duplicates`,
     `missing`, `outliers`, …;
  3. sklearn: `DtkTransformer(op, **params)` (fit / transform, pandas in/out)
     and `workspace_pipeline(name)` → `Pipeline`, usable in `cross_val_score`.
- **Transform protocol** (replaces `fn(df, params, fit)`):
  `@transform(op, params_model=...)` registers `fit(df, params) -> state`
  (JSON-safe dict) and `apply(df, params, state) -> df`. Replay: `both` → fit
  on train as of that step, apply to train and test; `train` / `test` → fit and
  apply on that role. Strict pydantic params → the front builds forms
  generically. Ops split by file: `ops/transforms/{cleaning,impute,encode,scale,features}.py`.
- **Export** (MAT-47): processed parquet + `manifest.json` (source hashes,
  steps with fitted states, versions) — raw inputs never modified.

## Layers

```
dtk_engine  ──  contract (JSON)  ──  EngineClient  ──  front (dtk_streamlit today, web tomorrow)
```

| Layer | Repo · path | May import |
|---|---|---|
| Engine | `datatoolkit` · `src/dtk_engine/` | pydantic, pandas, plotly, stdlib. **Never** a front. |
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
```

Errors (`dtk_engine/errors.py`): `UnknownKeyError` (unknown key id),
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
`filter_records` in `render.py`). `Result.show()` for notebooks (imports plotly
lazily, prints metrics, renders figures; no front dependency). Since the
contract returns a dict, the notebook idiom is
`Result(**run_key("train_test_check", {})).show()`.

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

`EngineClient` is a `typing.Protocol` with the three contract methods.
`LocalClient` calls `dtk_engine.contract` in-process. `HttpClient` = future.
Switching front = implement `HOWTO/add-a-front.md`, touch nothing in the engine.

## Engine internals: sources → ops → keys

Principle: **one generalized input → one output.** Behind the unchanged contract
the engine splits into three internal layers:

```
dtk_engine/
  sources/   SourceSpec (JSON) ──load()──▶ pd.DataFrame        generalized input
  ops/       pure functions: DataFrame(s) ─▶ DataFrame / dict   reused by keys and pipelines
  keys/      thin: Params(sources…) → load → ops → Result      one output
  (pipeline/ later)
```

- `sources/` never knows about keys; `ops/` never knows about `Result` or
  pydantic (testable alone); `keys/` only glue.
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

Files: `sources/spec.py` (`CsvSource`, `DatasetSource`, the `SourceSpec` union
— **add every new reader's spec to this union, and file readers also to
`FileSourceSpec`**, the file-only union a workspace uses for X / y so it cannot
reference a `dataset` source), `sources/registry.py` (`@reader`, `load`),
`sources/csv_pandas.py`, `sources/dataset.py`. Spec models are strict (`extra="forbid"`).

| `kind` | Reader | Status |
|---|---|---|
| `csv` | pandas `read_csv`: `path`, `sep` (`"auto"` → `csv.Sniffer` on the first 64 KB among `,` `;` tab `\|`, then C engine; fallback `sep=None, engine="python"`), `encoding`, `decimal`, `header` (null = no header). A Windows `path` (`C:\…`, `C:/…`) that does not exist is mapped to `/mnt/<drive>/…` on Linux/WSL. A single-column file (no candidate separator in the header) reads with `,` | done |
| `dataset` | current state of a workspace dataset: `workspace`, `role` (`train` \| `test`), `labeled` (default true) → load X, join y (or keep / drop `target_column` per `labeled`), replay the steps for the role | done |
| `csv_robust` | malformed CSVs (bad lines, mixed separators, junk headers) | later |
| `parquet` | polars or duckdb backend → pandas | later, needs approval |
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

## Workspace: loaded datasets + transform log (built 2026-09-25, no transform op yet)

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
- **Transform signature**: `@transform(op)` registers `fn(df, params, fit) -> df`.
  `fit` is None when applied to train or by a test-only step; for a `both` step
  applied to test, `fit` is the train frame as of that step (earlier train /
  both steps already replayed). Unknown op → `SourceError`, raised before any
  work. No op registered yet.
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
