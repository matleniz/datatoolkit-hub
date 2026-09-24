# How to add a key (3 files, zero front code)

1. **Code** — `packages/engine/src/dtk_engine/keys/<id>.py`: a pydantic
   `Params` (defaults = runnable demo) + a `@key(...)`-decorated
   `run(params) -> Result`. Import the module in `keys/__init__.py`.
2. **Test** — `tests/keys/test_<id>.py`: run with default params, assert the
   metrics that matter. The generic `tests/test_contract.py` already checks every
   registered key returns JSON-serializable output.
3. **Card** — hub `KEYS/<id>.md` from `KEYS/_template.md`, plus one row in
   `INDEX.md` → Keys table. (Worker: propose it via `propose-doc-change`; the
   coordinator writes the hub.)

Rules: any new dependency → `STACK.md` + Matteo's approval first. Keys return
data, never render. `fleet gate` must be clean before the PR.
