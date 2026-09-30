# Front web — datatoolkit Studio (repo `datatoolkit-web`)

Validated 2026-09-27 (MAT-126). Behaviour spec = the interactive prototype:
https://claude.ai/artifact/642ZsyQkeLDfNsifJAXdHQ (source kept locally in
`documents/studio-prototype.dc.html`, and in the web repo as
`docs/prototype.dc.html`). When this page and the prototype disagree, the
prototype wins for UX, this page wins for the contract.

## Stack
React + TypeScript (strict) + Vite, ESLint, Vitest (unit), Playwright (e2e with
screenshots). No state library (React context + reducer), no grid library (the
grid is ours, like the prototype). Talks only to the engine HTTP API (`dtk-api`,
below); never imports the engine.

## Screens
1. **Sources** — workspaces list; files with a role each (Train X, Train y, Test X,
   Merge, Ignore; guessed from names, never applied without the user); target (y
   file joined by order / key, or a column of train X); merge (key, also into
   test); result columns coloured by origin (X / y / merged); engine errors shown
   verbatim. **Workspace manager** (sidebar, MAT-171): per-workspace name,
   last modified, train/test summary, step count, target; Rename, Duplicate
   (shares content-addressed source refs, not copied), Delete (confirmation,
   multi-select), Export shortcut; search/sort; deleting the active workspace
   falls back cleanly to another one (MAT-149 race).
2. **Train / test alignment** — `align_report`: one row per column (match, type
   mismatch, value mismatch, missing in test, extra in test, label), train/test
   means. A `value_mismatch` row (categorical values unseen in test) is "to
   decide" only when `blocking` (near-match spelling variants, or too large a
   share of test rows affected) — otherwise it stays visible but informational
   (rare new categories that one-hot `handle_unknown` absorbs, MAT-179). Fixes
   are explicit clicks: source option (e.g. csv `decimal=","`), Map on test for
   value mismatches, test-only steps (`rename`, `cast`, `drop_columns` with
   `target: test`) flagged as alignment steps and kept first in the pipeline.
   Removable one by one.
3. **Workbench**
   - **Pipeline bar**: one node per version (op, sub-label, shape, delta, target,
     fitted dot, stage colour = course stage); click = time travel (read-only);
     × = remove step and replay; failing step shown red with the engine message;
     `+ Step` opens the step picker (also lists `polynomial`, `power_transform`,
     `quantile_transform`, `spline` under Encode & transform). The same picker
     opens from the tool rail's **Transform** entry (MAT-233).
   - **Grid**: header = name, kind chip, mini histogram / top values, missing bar,
     up to 2 alerts; cells coloured missing / sentinel / outlier; preview colours
     changed / new / removed. Click header (shift or multi toggle = add), row
     number, cell. **Right-click header** = column menu (inspect, add to
     selection, compare, distribution, type-relevant transforms, rename, cast,
     use in formula, set / unset target, drop, Chart…).
   - **Inspector** (right): column profile + Analyse / Transform buttons; row
     inspector; cell → rule (replace as missing, map value); multi-selection →
     compare, correlation, derive, **New feature…** (formula editor: selected
     columns as chips, function palette with one-line help, autocomplete on
     columns/`@variables` and `np.` / `numpy.` functions, non-identifier
     column names inserted as `df["…"]`, a **Python** section of insertable
     examples (`1 if Age < 18 else 0`, `np.where(…)`, `18 <= Age < 65`) and
     each function's numpy form in its help (MAT-241) — stored workspace
     variables still load and replay,
     but Studio no longer creates them, MAT-231 — live preview, inline engine
     errors, output name),
     **Polynomial features…** / **Power transform…** / **Quantile
     transform…** (open the step editor prefilled with the numeric selection;
     blocked with an "Impute missing values first" hint when the selection has
     missing values, MAT-191), scale, drop, Chart.
   - **Step editor** (replaces the inspector): what the op does + course ref,
     every param editable (generated from `transform_schema` + column hints),
     Apply to train / train+test / test, **Learned on train** (fitted state
     from `preview_step`), effect (diff counts), live preview on the grid,
     Apply / Discard. Nothing changes without Apply.
   - **Left panel = Suggestions** only (MAT-231; the Variables and Recipe tabs
     were removed — the pipeline bar already does time travel): analysis
     keys' findings, each only *opens the editor*.
   - **Collapsible side panels** (MAT-232): left panel and inspector each
     collapse to a 32 px strip (chevron); state in `AppState.panels`, mirrored
     to `localStorage["dtk.panels"]` (per browser). Collapsing the right panel
     is blocked while the step editor is open (no unapplied edit is hidden).
   - **Tool rail + dock**: **Transform** (opens the step picker, prefilled from
     the grid selection; never a dock window, MAT-233), Compare, Correlation,
     Distribution, Missing, Outliers, Target, Train vs test, **Chart** —
     windows bound to the grid selection, on a snap-to-grid layout
     (react-grid-layout, MAT-234): drag the title bar to move, the corner or
     right / bottom edge to resize, no overlap (windows float up); bottom (12
     columns) and right (2 columns) docks keep separate layouts; S / M / L set
     the dock's height (bottom) or width (right) and the grid scales with it;
     maximize; Plotly figures re-lay out once a resize ends. Open windows,
     position, size and layouts persist per workspace in browser storage
     (`dtk.dock.<workspace>`) and survive a reload. No keyboard move (the old
     reorder arrows and "wide" toggle are gone).
     **Analysis window shell** (MAT-235, generic, driven by the `Result`; all
     windows except Compare's native matrix and Chart): one compact chrome
     line (role · source, scope chip, Split by for Distribution, Parameters
     chip, "bound to…" truncated with tooltip — MAT-246) → `Result.headline`
     (1 line in short windows, 2 otherwise) → the selected figure filling the
     window (min 180 px; the default dock height fits 3 windows with full
     axes, no scroll) → view tabs (one per figure + Table; default = the
     `main: true` figure, else the first; choice kept per window in
     `toolViews`, in memory) + front-only display controls (sort / top-N,
     count vs %, log — rewrite the Plotly JSON, never recompute) + **Open in
     Chart** (prefills Chart via `chartPrefill.ts`) → collapsed **Details**
     drawer (metric tiles, sortable paginated tables). Clicking a bar or
     heatmap cell named after a column selects it in the grid. Plotly mode
     bar (PNG/SVG) on every figure, hover only; figure colours follow
     `tokens.css`. **Parameters** panel (per window,
     MAT-174): schema-driven knobs for the bound key, generated from
     `GET /keys/{id}/schema` (same `schemaToFields` mapping as the step
     editor); changing a param re-runs the key (debounced); **Reset to
     defaults** uses the schema defaults plus per-column `suggested_params`
     from `column_profiles`; values persist in Studio UI state per window (and
     per column for Distribution / Outliers / Target). Distribution always
     runs the `column_distribution` key (not the profile mini-histogram) so
     bins/log/norm apply; Correlation runs `correlations` (method/threshold).
     **Chart** (MAT-172, redesigned MAT-240): a grid of icon tiles
     (histogram, box, violin, bar/count, scatter, line, heatmap/density, pie,
     scatter matrix) — tiles that do not fit the selection are greyed with
     the reason on hover, the recommended one is marked; one compact X / Y /
     Color row with a **by target** pill; facets / size / agg / bins /
     sample / trendline / log folded under **More**; the figure fills the
     window. Prefilled from the grid selection; runs the engine `chart` key
     (Plotly mode bar for PNG/SVG export). Opens larger than other windows by
     default (MAT-252); any newly opened window is placed where it is visible,
     shrinking to the free width (min 4 columns) rather than below the fold. Saved chart specs (name + params)
     reopen and re-render on the current pipeline version (persisted in
     browser storage until the engine grows `Workspace.charts`, MAT-185).
   - **Export**: workspace JSON + `export_workspace` outputs, leak line
     ("n fitted steps learned on train, nothing refitted on test").

## HTTP API (`dtk_engine/http.py`, extra `api`, MAT-130)
Base `/api`. Bodies and responses are the contract's JSON, unchanged.

| Route | Contract |
|---|---|
| `GET /keys` · `GET /keys/{id}/schema` · `POST /keys/{id}/run` `{params}` | `list_keys`, `key_schema`, `run_key` |
| `GET /transforms` · `GET /transforms/{op}/schema` | `list_transforms`, `transform_schema` |
| `GET /workspaces` · `GET/PUT/DELETE /workspaces/{name}` | store |
| `GET /workspaces/summaries` | `list_workspace_summaries` (MAT-171) |
| `POST /workspaces/{name}/rename` `{new_name}` | `rename_workspace` (MAT-171) |
| `POST /workspaces/{name}/duplicate` `{new_name}` | `duplicate_workspace` (MAT-171) |
| `POST /workspaces/{name}/export` `{out_dir, overwrite}` | `export_workspace` |
| `POST /source/columns` `{spec}` | `source_columns` |
| `POST /workspace/preview` `{workspace, role, head_rows}` | `preview_workspace` |
| `POST /workspace/rows` `{workspace, role, version, offset, limit, columns?}` | `workspace_rows` (MAT-127; optional `columns` filter MAT-152) |
| `POST /workspace/profiles` `{workspace, role, version, columns?}` | `column_profiles` (optional `columns` filter MAT-152) |
| `POST /workspace/preview-step` `{workspace, step, role}` | `preview_step` |
| `POST /workspace/align` `{workspace}` | `align_report` |
| `PUT /uploads/{filename}` (raw body) → `{path}` | content-addressed under `$DTK_UPLOAD_DIR` |

Errors: `{type, message}`; 404 for `UnknownKeyError`, `UnknownTransformError`,
`WorkspaceNotFoundError`; 422 for `KeyParamsError`, `SourceError`. A params
validation failure gives a concise `message` (`<loc>: <msg>; …`, never the
pydantic dump) plus `details: [{loc, msg, type}]` (MAT-142). CORS allows the
Vite dev origin. `dtk-api --port 8765` runs uvicorn.

The front builds each key's params from `GET /keys/{id}/schema` (only the
properties the key declares, e.g. `test` / `target`), and dedupes identical
in-flight requests; one rows page + one profiles call per (steps, role,
version), workspace `PUT` only when its JSON changed (MAT-144).

## Run it
`cd ~/datatoolkit && uv run --extra api dtk-api --port 8765`, then in
`~/datatoolkit-web`: `npm run dev` (proxies `/api`). Default export directory:
`$DTK_HOME/exports/<workspace>`. Docker (no dev tools; nginx replaces the
Vite proxy, Studio on :8080): `HOWTO/run-with-docker.md`.

## Tests
Unit (Vitest) for pure logic: selection, diff colouring, window layout
reducer (dock grid: `dockLayout.ts`, MAT-234), schema → editor fields. E2E (Playwright, `npm run e2e`): starts
`dtk-api` + Vite, drives the six flows (sources, alignment, workbench edit,
formula (stored `@variables` replay), compare + windows, export) on the prototype's dirty churn
fixtures and on a real dataset, one screenshot per key state in
`docs/screenshots/t1-e2e/<flow>/`. Flow 7 (real Parkinson files, skipped if absent)
also asserts the workbench grid is ready in < 8 s with no duplicate
`POST /workspace/*` request. Captures wait for real content
(`waitForGridReady`), never a loading state.

## State (2026-09-30)
Built and merged: engine datatoolkit #30–#63, web datatoolkit-web #1–#48
(#40–#44 = the 2026-09-29 UX review, MAT-230: Suggestions-only left panel,
collapsible panels, Transform in the rail, grid dock, figure-first window
shell + compact chrome; engine #56 `plotly_lock`, #57 `headline` / `main`).
58 e2e specs on main (10 flows + per-ticket specs: MAT-149, 152, 154, 155,
160, 167, 169, 171, 173, 174, 177, 231/232, 233, 234, 235); flow2 / flow3
(alignment report not ready within 5 s) and flow7 / mat171 (order / cold
cache) can flake, MAT-222 / MAT-245 — they pass on re-run. Workbench open on 55 603 rows: ~1.5 s.
Known gaps: the export outputs list scrolls rather than showing all lines at
once; saved chart specs live in browser storage until the engine gains
`Workspace.charts` (MAT-185); `GET /workspaces/summaries` always returns
`shape: null` (perf fix MAT-200, lazy/cached shape tracked as MAT-204).
