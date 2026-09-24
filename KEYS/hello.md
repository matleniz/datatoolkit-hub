# Key `hello` — Hello

- **Category:** demo
- **Code:** `packages/engine/src/dtk_engine/keys/hello.py` · test `tests/keys/test_hello.py`
- **What it answers:** nothing — plumbing check for the whole chain (engine → contract → client → front).

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `n` | int, 1..1000 | 10 | number of points |

## Result
- metrics: `n`, `sum` (sum of squares 0..n-1)
- tables: `squares` — records `{x, y}` with y = x²
- figures: `squares` — Plotly line of y = x²

## Notes
Delete or keep as a smoke test once real keys exist.
