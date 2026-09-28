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
   verbatim.
2. **Train / test alignment** — `align_report`: one row per column (match, type
   mismatch, missing in test, extra in test, label), train/test means. Fixes are
   explicit clicks: source option (e.g. csv `decimal=","`), test-only steps
   (`rename`, `cast`, `drop_columns` with `target: test`) flagged as alignment
   steps and kept first in the pipeline. Removable one by one.
3. **Workbench**
   - **Pipeline bar**: one node per version (op, sub-label, shape, delta, target,
     fitted dot, stage colour = course stage); click = time travel (read-only);
     × = remove step and replay; failing step shown red with the engine message;
     `+ Step` opens the step picker.
   - **Grid**: header = name, kind chip, mini histogram / top values, missing bar,
     up to 2 alerts; cells coloured missing / sentinel / outlier; preview colours
     changed / new / removed. Click header (shift or multi toggle = add), row
     number, cell. **Right-click header** = column menu (inspect, add to
     selection, compare, distribution, type-relevant transforms, rename, cast,
     new variable, use in formula, set / unset target, drop).
   - **Inspector** (right): column profile + Analyse / Transform buttons; row
     inspector; cell → rule (replace as missing, map value); multi-selection →
     compare, correlation, derive, formula, scale, drop.
   - **Step editor** (replaces the inspector): what the op does + course ref,
     every param editable (generated from `transform_schema` + column hints),
     Apply to train / train+test / test, **Learned on train** (fitted state
     from `preview_step`), effect (diff counts), live preview on the grid,
     Apply / Discard. Nothing changes without Apply.
   - **Left tabs**: Variables (named train statistics `@name`, create / delete /
     insert), Suggestions (analysis keys' findings, each only *opens the
     editor*), Recipe (steps list, click = time travel).
   - **Tool rail + dock**: Compare, Correlation, Distribution, Missing,
     Outliers, Target, Train vs test — windows bound to the grid selection,
     reorder by drag or arrows, wide (2 columns), maximize, dock bottom / right,
     sizes S / M / L.
   - **Export**: workspace JSON + `export_workspace` outputs, leak line
     ("n fitted steps learned on train, nothing refitted on test").

## HTTP API (`dtk_engine/http.py`, extra `api`, MAT-130)
Base `/api`. Bodies and responses are the contract's JSON, unchanged.

| Route | Contract |
|---|---|
| `GET /keys` · `GET /keys/{id}/schema` · `POST /keys/{id}/run` `{params}` | `list_keys`, `key_schema`, `run_key` |
| `GET /transforms` · `GET /transforms/{op}/schema` | `list_transforms`, `transform_schema` |
| `GET /workspaces` · `GET/PUT/DELETE /workspaces/{name}` | store |
| `POST /workspaces/{name}/export` `{out_dir, overwrite}` | `export_workspace` |
| `POST /source/columns` `{spec}` | `source_columns` |
| `POST /workspace/preview` `{workspace, role, head_rows}` | `preview_workspace` |
| `POST /workspace/rows` `{workspace, role, version, offset, limit}` | `workspace_rows` (MAT-127) |
| `POST /workspace/profiles` `{workspace, role, version}` | `column_profiles` |
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
`$DTK_HOME/exports/<workspace>`.

## Tests
Unit (Vitest) for pure logic: selection, diff colouring, window layout
reducer, schema → editor fields. E2E (Playwright, `npm run e2e`): starts
`dtk-api` + Vite, drives the six flows (sources, alignment, workbench edit,
variables + formula, compare + windows, export) on the prototype's dirty churn
fixtures and on a real dataset, one screenshot per key state in
`e2e/screenshots/<flow>/`. Flow 7 (real Parkinson files, skipped if absent)
also asserts the workbench grid is ready in < 8 s with no duplicate
`POST /workspace/*` request. Captures wait for real content
(`waitForGridReady`), never a loading state.

## State (2026-09-28)
Built and merged: engine datatoolkit #30–#34, web datatoolkit-web #1–#9;
7/7 e2e flows green. Workbench open on 55 603 rows: ~1.5 s. Known gap: the
export outputs list scrolls rather than showing all lines at once.
