# How to add a transform op

A transform is a workspace step (and an sklearn `DtkTransformer`). Protocol:
`ARCHITECTURE.md` → Workspace → Transform protocol. Reference op:
`src/dtk_engine/ops/transforms/cleaning.py::drop_columns`.

1. **Pick the family module** in `src/dtk_engine/ops/transforms/`
   (`cleaning`, `impute`, `encode`, `scale`, `features`, `selection`). Never
   edit `__init__.py` for an op in an existing module (it imports them all).
   A supervised op (fit reads the target) passes `needs_target=True` and has a
   `target` param.
2. **Params** — `class XParams(TransformParams)` (strict pydantic, one
   `Field(description=...)` per param: the front builds its form from it).
3. **fit** (only if the op learns something) — `def _fit(df, params) -> dict`
   returns a JSON-safe dict (numbers, strings, lists — no numpy, no pickle).
   It sees train only when the step targets `both`.
4. **apply** — `@transform("x", params_model=XParams, fit=_fit, title="...")`
   on `def x(df, params, state) -> pd.DataFrame`: return a new frame, never
   mutate `df`; first docstring line = description.
5. **Tests** (`tests/ops/transforms/test_<family>.py`): via `DtkTransformer`
   (fit on train, transform test uses train state) and via workspace replay
   on `both`.
6. **Hub catalog** — add the op to the hub `TRANSFORMS.md` (and its row in
   `CAPABILITIES.md` if a status changes): a worker files it with
   `propose-doc-change`, the coordinator writes it.

Zero front code: `list_transforms` / `transform_schema` expose it.
