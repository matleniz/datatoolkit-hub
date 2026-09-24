# Architecture — datatoolkit

Trusted-fact doc. Code in `~/datatoolkit`. If the code contradicts this file,
flag it (`propose-doc-change`), do not silently diverge.

## Layers

```
dtk_engine  ──  contract (JSON)  ──  EngineClient  ──  front (dtk_streamlit today, web tomorrow)
```

| Layer | Package / path | May import |
|---|---|---|
| Engine | `packages/engine/src/dtk_engine/` | pydantic, pandas, plotly, stdlib. **Never** a front. |
| Client | `packages/front-streamlit/src/dtk_streamlit/client.py` | `dtk_engine.contract` only |
| Front | `packages/front-streamlit/src/dtk_streamlit/` (all other modules) | `dtk_streamlit.client`, streamlit, plotly. **Never** `dtk_engine`. |

Enforced mechanically: ruff `TID251` bans `dtk_engine` in the front except
`client.py`; `tests/test_front_isolation.py` asserts the same. Both run in
`fleet gate`.

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
```

Errors (`dtk_engine/errors.py`): `UnknownKeyError` (unknown key id),
`KeyParamsError` (params fail validation — incl. a misspelled param or an
unknown source `kind`), `SourceError` (a source cannot be loaded: missing /
empty / unreadable file; raised unchanged by `run_key`). The Streamlit front
shows any engine error via `st.error`.

An HTTP API later exposes exactly these three (`GET /keys`,
`GET /keys/{id}/schema`, `POST /keys/{id}/run`) with zero per-key code.

## Result (engine output)

`dtk_engine/result.py`, pydantic model:

| Field | Type | Meaning |
|---|---|---|
| `metrics` | `dict[str, float \| int \| str]` | headline numbers |
| `tables` | `list[Table]` — `{"title": str, "records": list[dict]}` | tabular output |
| `figures` | `list[Figure]` — `{"title": str, "plotly": dict}` | Plotly figure JSON (`json.loads(fig.to_json())`) |
| `text` | `str` | markdown commentary, may be empty |

Helpers: `Result.add_figure(title, fig)`, `Result.add_table(title, df)`
(convert to JSON-safe records). `Result.show()` for notebooks (imports plotly
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

Files: `sources/spec.py` (`CsvSource`, the `SourceSpec` union — **add every new
reader's spec to this union**), `sources/registry.py` (`@reader`, `load`),
`sources/csv_pandas.py`. Spec models are strict (`extra="forbid"`).

| `kind` | Reader | Status |
|---|---|---|
| `csv` | pandas `read_csv`: `path`, `sep` (`"auto"` → `csv.Sniffer` on the first 64 KB among `,` `;` tab `\|`, then C engine; fallback `sep=None, engine="python"`), `encoding`, `decimal`, `header` (null = no header). A Windows `path` (`C:\…`, `C:/…`) that does not exist is mapped to `/mnt/<drive>/…` on Linux/WSL | done |
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
`tests/front/test_schema_form.py`.

## Direction: transforms & pipelines (NOT built — design to be discussed first)

Capability families and backlog: `CAPABILITIES.md`.

- **Catalog**: named tables (`X_train`, `y_train`, `X_test`…), each from a SourceSpec.
- **Transform op**: table(s) + params → table (join, concat, derived feature, cast, drop, filter).
- **Check op** (= analysis key): table(s) → `Result`, produces no table.
- **Pipeline**: JSON list of steps `{op, inputs, params, output}` over the catalog.
  Reproducibility target: pipeline JSON + input file hashes + versions →
  deterministic run producing output tables + a run manifest.
