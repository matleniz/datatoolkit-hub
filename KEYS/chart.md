# Key `chart` — Chart

- **Category:** analysis
- **Code:** `src/dtk_engine/keys/chart.py` · test `tests/keys/test_chart.py`
- **What it answers:** Build a custom Plotly Express figure from typed params
  on a source frame (workspace version/role via `DatasetSource`, MAT-175).

## Params
| Name | Type | Default | Meaning |
|---|---|---|---|
| `source` | SourceSpec | demo train CSV | Frame to plot (`DatasetSource.version` honoured — time travel) |
| `chart` | enum | `histogram` | `histogram`, `box`, `violin`, `bar`, `count`, `scatter`, `line`, `heatmap`, `density_heatmap`, `pie`, `scatter_matrix` |
| `x`, `y`, `color`, `facet_row`, `facet_col`, `size` | column | `x="Age"` | Plotly Express aesthetics |
| `columns` | list[str] | `[]` | `scatter_matrix` dimensions (empty = first numeric columns) |
| `agg` | `count`\|`mean`\|`sum`\|`median`\|null | null | Aggregation for `bar` / `line` |
| `trendline` | bool | `false` | OLS overlay on `scatter` (numpy `polyfit`; no statsmodels) |
| `log_x`, `log_y` | bool | `false` | Log axes |
| `bins` | int | `30` | Histogram / density bins |
| `sample_size` | int\|null | `10000` | Engine-side row cap before plotting (sampling never happens in the browser) |

## Result
- metrics: `chart`, `n_rows`, `n_rows_source`, `n_sampled_out`, `trendline`
- figures: one Plotly figure JSON (`Result.add_figure`). A numeric / bool `color` with ≤ 10 distinct values (e.g. a binary target) is treated as categorical: discrete legend, one colour per value, consistent with trendline groups (MAT-251); high-cardinality numeric colours keep a continuous scale.
- text: a sampling note when rows were dropped

## Notes
- The Studio Chart tool window (MAT-172 front, `FRONT-WEB.md` → Workbench →
  Tool rail + dock) is the UI on top of this key; it prefills type/x/y from
  the grid selection and renders the returned figure via the same renderer
  used elsewhere for `Result.figures`. Saved chart specs are a front concern
  (currently browser storage, see MAT-185 for engine-side persistence).
- OLS trendline uses `numpy.polyfit`, not `statsmodels` (not on `STACK.md`).
- Notebook door: `api.chart(df, chart=..., **params)`.
