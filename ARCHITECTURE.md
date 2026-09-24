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
    # [{"id": "hello", "title": "Hello", "category": "demo", "description": "..."}]
def key_schema(key_id: str) -> dict
    # JSON Schema of the key's Params (pydantic model_json_schema())
def run_key(key_id: str, params: dict) -> dict
    # Result.model_dump(mode="json"); invalid params -> raises KeyParamsError
```

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
`Result(**run_key("hello", {"n": 5})).show()`.

Keys use absolute imports (`from dtk_engine.registry import key`): ruff TID252
forbids relative parent imports.

## A key

```python
# dtk_engine/keys/<id>.py
class Params(BaseModel): ...          # typed, defaults = sensible demo
@key(id="<id>", title="...", category="...", description="...")
def run(params: Params) -> Result: ...
```

`registry.py` holds the `@key` decorator and the registry; `keys/__init__.py`
imports every key module so registration happens on `import dtk_engine`.
Recipe: `HOWTO/add-a-key.md`.

## Front (generic, zero per-key code)

`dtk_streamlit/app.py`: sidebar radio over `client.list_keys()`, labelled
`category / title`, sorted by category → `render.form_from_schema(schema)` builds widgets from JSON Schema
(integer, number, string, boolean, enum; unknown types → JSON text input) →
`client.run_key` → `render.result(result_dict)` shows metrics, tables
(`st.dataframe`), figures (`st.plotly_chart(plotly.io.from_json(...))`), text.

`EngineClient` is a `typing.Protocol` with the three contract methods.
`LocalClient` calls `dtk_engine.contract` in-process. `HttpClient` = future.
Switching front = implement `HOWTO/add-a-front.md`, touch nothing in the engine.
