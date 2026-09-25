# How to add (or switch to) a front

A front is a renderer of the contract in `ARCHITECTURE.md`; it knows nothing
about individual keys.

1. Get an `EngineClient`: `LocalClient` (Python front, in-process) or
   `HttpClient` over the HTTP API (non-Python front — needs the API issue done).
2. Implement three generic views:
   - **catalog** from `list_keys()` (group by `category`)
   - **form** from `key_schema(id)` (JSON Schema → widgets)
   - **result** from `run_key(id, params)`: metrics, tables (`records`),
     figures (Plotly JSON — every Plotly binding renders it as-is), text (markdown)
3. Never import `dtk_engine` outside the client. A new front lives in its own
   repo (like `datatoolkit-streamlit`), depends on the engine via git, and
   copies the ruff `TID251` ban + `tests/test_front_isolation.py`.

If a key needs something the result views can't show, extend `Result` (engine +
this doc), never special-case the key in a front.
