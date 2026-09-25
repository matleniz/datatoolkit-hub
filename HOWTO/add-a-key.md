# How to add a key (3 files, zero front code)

1. **Code** — `src/dtk_engine/keys/<id>.py`: a
   `Params(KeyParams)` (strict; defaults = runnable demo — for data keys, a
   `source: SourceSpec` defaulting to `dtk_engine.demo_data`) + a `@key(...)`-decorated
   `run(params) -> Result`. Import the module in `keys/__init__.py`.
2. **Test** — `tests/keys/test_<id>.py`: run with default params, assert the
   metrics that matter. The generic `tests/test_contract.py` already checks every
   registered key returns JSON-serializable output.
3. **Card** — hub `KEYS/<id>.md` from `KEYS/_template.md`, plus one row in
   `INDEX.md` → Keys table. (Worker: propose it via `propose-doc-change`; the
   coordinator writes the hub.)

Keys stay thin: `load(source)` → functions from `dtk_engine/ops/` (pure,
no `Result`) → `Result`. Put reusable logic in `ops/`, not in the key.

Rules: any new dependency → `STACK.md` + Matteo's approval first. Keys return
data, never render. `fleet gate` must be clean before the PR.
