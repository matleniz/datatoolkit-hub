# Front web — datatoolkit Studio (repo `datatoolkit-web`)

Validated 2026-09-27 (MAT-126). Behaviour spec = the interactive prototype:
https://claude.ai/artifact/642ZsyQkeLDfNsifJAXdHQ (source kept locally in
`documents/studio-prototype.dc.html`, and in the web repo as
`docs/prototype.dc.html`). The prototype predates the 2026-09-29 UX review
(MAT-230: Suggestions-only left panel, grid dock, window shell); when it
disagrees with this page, this page wins.

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
     × = remove step and replay; ✎ (or right-click → **Edit step**) reopens
     the step editor pre-filled with the step's op, target and params, the
     view pinned to the step's input version and the live preview on the
     steps before it; Apply (enabled once something changed) replaces the
     step at its index, keeps the later steps and replays at the latest
     version; Discard returns to the latest version (datatoolkit-issues#10);
     ↶ / ↷ (and Ctrl/Cmd+Z, Shift+Ctrl/Cmd+Z or
     Ctrl+Y outside text fields) undo / redo changes to the steps (add, edit, remove,
     alignment steps), replayed at the latest version; in-memory history (100
     levels), reset on loading another workspace, disabled while a step is
     being edited (datatoolkit-issues#16; revert-to-version and disable-a-step
     not built); failing step shown red with the engine message;
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
     columns / `@variables`, function names (incl. `group_mean` /
     `group_prev` / `group_interp`, also in the palette) and `np.` / `numpy.`
     functions, the engine error shown right under the Expr field (red
     outline; also for `impute(strategy=formula)`, datatoolkit-issues#12),
     non-identifier
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
     every param editable (generated from `transform_schema` + column hints;
     a param carrying `x-dtk-when` is shown, validated and sent only while
     every listed sibling has one of its values; an empty `x-dtk-semantic`
     param such as `impute.by` / `ffill.by` is prefilled with the frame's only
     column of that `semantic`, datatoolkit-issues#48),
     Apply to train / train+test / test, **Learned on train** (fitted state
     from `preview_step`), effect (diff counts), live preview on the grid,
     Apply / Discard. Nothing changes without Apply.
   - **Left panel = Suggestions** only (MAT-231; the Variables and Recipe tabs
     were removed — the pipeline bar already does time travel): analysis
     keys' findings, each only *opens the editor*. Each card can be
     **dismissed** (datatoolkit-issues#15): browser-local, per workspace
     (`localStorage["dtk.dismissedSuggestions.<workspace>"]`); the id hashes the
     key, the suggested step (op / target / params) and the finding (column,
     title, detail), so the same finding stays hidden after an unrelated step
     and shows again once its content changes. "Show dismissed (N)" lists hidden
     cards with **Restore**; the badge counts non-dismissed cards.
   - **Collapsible side panels** (MAT-232): left panel and inspector each
     collapse to a 32 px strip (chevron); state in `AppState.panels`, mirrored
     to `localStorage["dtk.panels"]` (per browser). Collapsing the right panel
     is blocked while the step editor is open (no unapplied edit is hidden).
   - **Tool rail + dock**: **Transform** (opens the step picker, prefilled from
     the grid selection; never a dock window, MAT-233), Compare, Correlation,
     Distribution, Missing, Outliers, Target, Train vs test, Feature
     selection, **Chart** —
     windows bound to the grid selection, on a snap-to-grid layout
     (react-grid-layout, MAT-234): drag the title bar to move, the corner or
     right / bottom edge to resize, no overlap (windows float up); bottom (12
     columns) and right (2 columns) docks keep separate layouts; S / M / L set
     the dock's height (bottom) or width (right) and the grid scales with it;
     maximize; Plotly figures re-lay out once a resize ends. Open windows,
     position, size and layouts persist per workspace in browser storage
     (`dtk.dock.<workspace>`) and survive a reload. No keyboard move (the old
     reorder arrows and "wide" toggle are gone). Each window (and the
     Suggestions tab) shows the frame its numbers come from (`train · v2`,
     `test · sources`, `IdentityStrip`); while a step is being edited it shows
     the last applied version (the live preview is grid-only).
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
     reopen and re-render on the current pipeline version (stored on the
     engine workspace, `Workspace.charts`, MAT-185, datatoolkit-issues#11:
     Save chart PUTs the workspace and shows the engine's 422 `duplicate
     chart name` with a **Replace** action; charts are left out of frame /
     analysis requests and of the data identity; charts saved in browser
     storage by older builds are migrated once).
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
| `PUT /uploads/{filename}` (raw body) → `{path}` | content-addressed under `$DTK_UPLOAD_DIR` (default `$DTK_HOME/uploads`) |
| `PUT/GET /ui/context` · `GET /ui/events?session=&token=` (SSE) · `POST /ui/ack` · `POST /ui/commands` `{type, …}` (`?session=&timeout=`) | UI bridge, not the contract (`dtk_engine/ui_bridge.py`, datatoolkit-issues#62; `AGENT-BRIDGE.md`) |

Errors: `{type, message}`; 404 for `UnknownKeyError`, `UnknownTransformError`,
`WorkspaceNotFoundError`; 422 for `KeyParamsError`, `SourceError`. A params
validation failure gives a concise `message` (`<loc>: <msg>; …`, never the
pydantic dump) plus `details: [{loc, msg, type}]` (MAT-142). CORS allows the
Vite dev origin (localhost / 127.0.0.1 :5173) plus `DTK_CORS_ORIGINS`
(comma-separated). `dtk-api --port 8765` runs uvicorn.

The front builds each key's params from `GET /keys/{id}/schema` (only the
properties the key declares, e.g. `test` / `target`), and dedupes identical
in-flight requests; one rows page + one profiles call per (steps, role,
version), workspace `PUT` only when its JSON changed (MAT-144).

**Agent UI bridge (datatoolkit-issues#62, #63).** The `/ui/*` routes need the
per-run token (`Authorization: Bearer`, or `?token=` for `EventSource`; engine
side `DTK_UI_TOKEN` or a random token at `app.state.ui_bridge.token`), an
allowed `Origin` when present and a loopback or `DTK_UI_ALLOWED_HOSTS` `Host`;
the contract routes stay token-free. Studio reads the token from
`<meta name="dtk-ui-token">` (dev: a Vite plugin when `DTK_UI_TOKEN` is set;
`dtk-studio` injects it, loopback Host only); no meta = bridge off. These calls
bypass the request dedupe. Modules: `src/state/uiContext.ts` (AppState →
published context), `src/state/agentCommands.ts` (parse, stale check, review,
apply, ack), `src/bench/agent/AgentBridge.tsx` (publisher, SSE listener, Undo
toast, review banner). Agent step edits go through the reducer action
`APPLY_STEP_BATCH {ops}` (ordered `add` / `replace` / `remove`, one undo
entry, no-op on an invalid index; helpers and `orderSteps` in
`src/state/stepOps.ts`). Semantics: `AGENT-BRIDGE.md` → "As built".

**Refresh identity (MAT-175).** Every consumer that shows data — grid,
profiles, inspector, dock windows, Chart, Suggestions — keys on one
*data identity* (`src/bench/dataIdentity.ts`): workspace name + role +
effective version + a hash of the sources, label join, merges and
`steps[0:version]` (params included). Nothing keys on `steps.length`: editing
a step's params refreshes everything, changing a step after the viewed
version refreshes nothing. Analysis keys get a `{kind: "dataset", workspace,
role, labeled, version}` source with the explicit effective version
(`DatasetSource.version`), so time travel and Train / Test apply to every
window. Suggestions analyse train (+ test) at the viewed version. Keys read
the named workspace
from the engine store, so consumers await a chained, non-debounced `PUT` of
the current workspace before `run_key` (`ensureWorkspaceSaved`,
`src/state/workspaceSaveGate.ts`). Train vs test (`train_test_check`) takes
`train` + `test` (no `source`); the target is only sent on labeled (train)
sources. Acceptance: `e2e/mat175-refresh-matrix.spec.ts`.

## Run it
`cd ~/datatoolkit && uv run --extra api dtk-api --port 8765`, then in
`~/datatoolkit-web`: `npm run dev` (http://localhost:5173, proxies `/api` to
:8765). Default export directory: `$DTK_HOME/exports/<workspace>` when train X
is an upload (`$DTK_HOME/uploads/…`), else `/tmp/exports/<workspace>`
(`src/bench/export/exportPaths.ts`). Docker (no dev tools; nginx replaces the
Vite proxy, Studio on :8080): `HOWTO/run-with-docker.md`. Share / no Docker:
`HOWTO/share-and-run.md` (`dtk-studio` serves the build + engine on :8080, same
origin `/api`).

## Tests
Unit (Vitest, `tests/**/*.test.ts`) for pure logic: selection, diff
colouring, dock grid layout (`src/state/dockLayout.ts`, MAT-234), schema →
editor fields, sources / alignment / workspace-manager logic, chart picker and
prefill, save gate. Gate (`fleet gate`): lint (ESLint, with
`sonarjs/cognitive-complexity` ≤ 25), typecheck, unit, `npx knip@6 --exclude
types`; the same checks run in `.github/workflows/ci.yml` (datatoolkit-issues#3).
E2E (Playwright, `npm run e2e`, chromium only, run separately): starts its
own `dtk-api` (:8766) + Vite (:5175) on a temp `DTK_HOME`, then runs flows
1–10 (sources, alignment, workbench edit, formula with stored `@variables`
replay, compare + windows, export, real data, column scope, distribution by,
chart) plus per-ticket specs on the prototype's dirty churn fixtures and real
datasets. Screenshots go to `e2e/screenshots/<flow>/` (gitignored); the
committed copies in `docs/screenshots/t1-e2e/<flow>/` are refreshed only with
`DTK_E2E_SCREENSHOTS=1`. Flow 7 (real Parkinson files, skipped if absent)
also asserts the workbench grid is ready in < 8 s with no duplicate
`POST /workspace/*` request. Captures wait for real content
(`waitForGridReady`), never a loading state.

## State (2026-10-01)
Built and merged: engine datatoolkit #30–#82, web datatoolkit-web #1–#77
(#40–#44 = the 2026-09-29 UX review, MAT-230: Suggestions-only left panel,
collapsible panels, Transform in the rail, grid dock, figure-first window
shell + compact chrome; engine #56 `plotly_lock`, #57 `headline` / `main`).
65 e2e tests in 39 spec files on main (flows 1–10, formats-sources and
per-ticket specs: MAT-149, 152, 154, 155, 160, 167, 169, 171, 173, 174, 175,
177, 205, 231/232, 233, 234, 235, 240, 241, 252, datatoolkit-issues #10, #15,
#16, #48); 308 unit tests. Specs wait on readiness signals (alignment report,
suggestions, grid identity, applied step) rather than fixed timeouts and pass
alone and in the full run (datatoolkit-issues#13); the e2e Vite config
pre-bundles every dependency so a cold cache does not reload the page
(datatoolkit-issues#14). Full run on a cold cache, 2026-10-01: 65/65, 5.4 min.
Deleting a workspace waits for the save already on the wire, then undoes it
(`src/state/workspaceSaveGate.ts`). Workbench open on 55 603 rows: ~1.5 s.
Known gaps: the export outputs list scrolls rather than showing all lines at
once.
`GET /workspaces/summaries` returns cached `train` / `test` `shape`
(MAT-204; content-addressed, no frame load when fresh).
