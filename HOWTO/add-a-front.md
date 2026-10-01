# How to add (or switch to) a front

A front is a renderer of the contract in `ARCHITECTURE.md`; it knows nothing
about individual keys.

1. Talk to the engine through the HTTP API (`dtk-api`, routes in
   `FRONT-WEB.md`) with one client module — Studio's is `src/api/client.ts`.
   A Python front may instead call `dtk_engine.contract` in-process, from a
   single client module.
2. Implement three generic views:
   - **catalog** from `list_keys()` (group by `category`)
   - **form** from `key_schema(id)` (JSON Schema → widgets)
   - **result** from `run_key(id, params)`: metrics, tables (`records`),
     figures (Plotly JSON — every Plotly binding renders it as-is), text (markdown)
3. Never import `dtk_engine` outside the client. A new front lives in its own
   repo (like `datatoolkit-web`). A Python front enforces the barrier with a
   ruff `TID251` ban on `dtk_engine` outside the client plus an isolation test
   (pattern of the archived `datatoolkit-streamlit` repo).

If a key needs something the result views can't show, extend `Result` (engine +
this doc), never special-case the key in a front.
