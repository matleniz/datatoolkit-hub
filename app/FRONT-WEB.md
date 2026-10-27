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
below); never imports the engine. Runtime libraries: `plotly.js-dist-min`
(figures), `react-grid-layout` (dock), `marked` + `dompurify` (agent Markdown,
sanitised), `@xterm/xterm` + `@xterm/addon-fit` (terminal panel); approvals in
`STACK.md`.

## Screens
1. **Sources** — workspaces list; files with a role each (Train X, Train y, Test X,
   Merge, Ignore; guessed from names, never applied without the user); target (y
   file joined by order / key, or a column of train X); merge (key, also into
   test); result columns coloured by origin (X / y / merged); engine errors shown
   verbatim. **Workspace manager** (sidebar, MAT-171): per-workspace name,
   last modified, train/test summary, step count, target; Rename, Duplicate
   (shares content-addressed source refs, not copied), Delete (confirmation,
   multi-select), Export shortcut; search/sort; deleting the active workspace
   falls back cleanly to another one (MAT-149 race). Sources edits autosave as
   they happen (datatoolkit-issues#234, web #152): a valid setup (no build
   errors, train file with its own columns) dispatches `SET_SOURCES_SPEC`, which
   replaces only `datasets`, `label` and `merges`; variables, memory,
   documents, `id_counters`, undo history and dock are untouched.
   Autosave failures (datatoolkit-issues#218, web #153): a failed PUT shows a
   persistent "Not saved: <engine message> — Retry" banner for that workspace
   only; it clears on a successful save or when the workspace returns to its
   saved value; leaving a workspace with a pending failure shows "edits to
   workspace … were lost" (Dismiss). `ensureWorkspaceSaved` now rejects, so
   analysis runs surface the error instead of using a stale workspace.
   Load feedback (datatoolkit-issues#246, #247, web #156): every failed load
   shows the engine's message (or the HTTP status) next to the source or in
   the preview; a busy indicator (`BusyIndicator`, `role="status"`, 150 ms
   show delay) with Cancel sits on the Sources panel, the step list and the
   grid while a request is in flight. Saves are never cancellable. Follow-ups:
   datatoolkit-issues#250.
   Frame cache (datatoolkit-issues#245, web #157): `src/bench/frameCache.ts`
   keeps the first page of rows, profiles and pipeline shapes per (workspace,
   role, version, hash of sources/params/merges/label/steps, grid view); LRU of
   8, 60 s TTL, errored or empty frames never cached; cleared on Retry, upload
   and when the tab becomes visible. Train↔test switches back send no data
   request. Engine side needed nothing (its path+mtime+size cache answers in
   8-35 ms at 50k rows).
   Duplicate, Rename and Export first await the pending autosave, so they
   never act on an older stored workspace (datatoolkit-issues#248, web #158).
   Startup fallback (datatoolkit-issues#220, web #160): if opening the
   remembered workspace fails (HTTP error, invalid schema, deleted), Studio
   forgets that name, opens `churn` or the first valid entry and shows a
   dismissible notice naming the skipped workspace and the engine's reason;
   if every candidate fails, the bootstrap error screen shows.
   Engine-down screens (datatoolkit-issues#237, #219, web #161): a failed
   startup shows a dedicated error screen (real error, hint per mode, Retry that
   re-runs the whole startup incl. the fallback above) and `<App/>` is not
   mounted until a workspace is loaded; the hint is chosen from
   `import.meta.env.DEV` only (dev: start dtk-api on 8765; built app:
   `docker compose logs engine` or the dtk-studio terminal), because Docker and
   uv serve the same production build on 8080. If the engine goes down later,
   the workspace list says "Engine unreachable" with Retry and a refetch on
   window focus (no polling) and "+ New workspace" is disabled.
   Role guessing (datatoolkit-issues#235, web #163): in a workspace where no
   file has a role yet, the first uploaded file is guessed Train X whatever its
   name (it used to be "Merge" for names like `customers*.csv`); a guess never
   overrides a role set by the user; the "pick a Train X / Train y" messages
   for a lone file are a calm hint, not red errors.
   Corrupt workspace at startup (datatoolkit-issues#236, web #164): when the
   remembered workspace is a corrupt file, the startup notice (same mechanism as
   #220) names it with the engine's reason and says to delete it or restore the
   file; the list shows the reason in clear for each corrupt entry (Delete
   enabled, Rename/Duplicate/Export disabled). The API returns only
   `{name, corrupt, error}`, so no file path is shown.
   Render errors and tab close (datatoolkit-issues#221, web #165): an
   `ErrorBoundary` sits OUTSIDE `AppProvider` in `src/main.tsx` (a reducer
   exception is thrown inside the provider, a boundary around `<App/>` would not
   catch it) and shows the error with Reload and "Export workspace JSON" (the
   last committed workspace of the save gate, fully try/caught, file
   `<name>.workspace.json`). On `pagehide` and `visibilitychange` to hidden the
   pending workspace is sent with `fetch(PUT, {keepalive: true})` (plain fetch
   above 60 KiB, browsers cap keepalive bodies near 64 KiB), chained on the
   save chain so later autosaves and delete/rename wait for it; it is not sent
   when nothing is pending, not twice for the same payload and not when a save
   already failed for that payload (#218). Known limit: at the exact moment of
   unload it can cross a chain PUT already on the wire (no version guard on the
   engine PUT; see the follow-up issue).
   Step groups, data layer (datatoolkit-issues#249 PR A, web #162; engine
   contract = datatoolkit PRs 163/164/165 and the contract comment on #249):
   `Workspace.groups` is a required field; every group is a contiguous slice of
   the step list (id, name 1-80 chars on one line, intent, `step_ids`, origin
   auto|manual, produced_by). `src/state/stepGroups.ts` is the single module
   holding the engine rules (`validateGroups`, `validateGroupName`,
   `newGroupId` = "g" + 8 base36 chars, never "g<digits>", singleton fill with
   id "g" + step id without its "s", and the 422 `detail{rule, group_id,
   step_id}` parser); every writer of `workspace.steps` (add, replace, remove,
   agent batches, align steps, undo/redo) goes through its helpers so steps and
   groups change together. Studio always sends `groups` in the workspace PUT
   and adopts the engine's normalised groups (`ADOPT_GROUPS`) without a new
   save; undo/redo snapshots include groups. No visible UI yet: the grouped
   step list is PR B, automatic and manual grouping PR C. An engine older than
   the contract refuses a file with `groups` (forward-only compatibility).
   Grouped step list (datatoolkit-issues#249 PR B, web #166): the pipeline
   list renders one row per group (`src/bench/pipeline/GroupBlock.tsx`,
   `groupRows.ts`), collapsed by default, keyboard expandable
   (`aria-expanded`), with name, intent, step count and a partial state when
   the selected version falls inside the group ("1 of 3 steps applied");
   selecting a group goes to its last step; deleting a group removes its steps
   and the group in ONE PUT (accessible confirm dialog `src/ConfirmDialog.tsx`, undoable; also used for clearing agent memory and deleting workspaces; deleting one step has no confirmation, it is undoable; web #181, datatoolkit-issues#265). Singleton
   groups (every existing workspace) render exactly like the former step rows.
   Expanded/collapsed is UI-only state. A group holding a failing step is
   forced open. There is no step reorder in the app (reordering can break
   replay): follow-up datatoolkit-issues#264.
   Grouping (datatoolkit-issues#249 PR C, web #167): all steps of one agent
   `propose_steps` batch land in ONE group (origin auto, produced_by agent;
   name = the proposal's group.name or a one-line name derived from the note,
   intent = group.intent or the note); no UI action adds several steps at once
   (verified by grep of every step-appending site, see the review of #167), so
   nothing else is auto-grouped. Manual: "Group rows..." (checkboxes, only
   adjacent rows; the new group is the union of the selected rows' steps) with an
   inline name field validated by `validateGroupName` BEFORE any PUT (one line,
   1-80 chars, no control characters or U+2028/2029), "Ungroup" (steps become
   singleton groups) and "Rename" (name and intent; origin becomes `manual`).
   Each operation is ONE valid PUT and is undoable. A group holding a failing
   step is forced open with its toggle disabled and explained. Polish items:
   datatoolkit-issues#275.
   Write-order guard (datatoolkit-issues#263, web #168; engine datatoolkit#175):
   every `PUT /workspaces/{name}` (autosave chain, keepalive flush, retries)
   carries `X-Dtk-Write-Seq`, a per-tab strictly increasing integer
   (`max(previous + 1, Date.now() * 1000)`, `src/api/writeSeq.ts`). The engine
   keeps the highest accepted value per workspace in memory and answers 409
   `StaleWriteError {detail:{rule:"stale_write", last_seq}}` to an older one
   without writing. Studio ignores a 409 that comes from its own newer PUT
   (silently, no banner, no retry); if `last_seq` is above everything this tab
   sent (clock moved backwards, another tab wrote) it raises its counter above
   `last_seq` and re-sends the latest payload ONCE. The unload flush is not
   re-sent. An engine without the guard ignores the header.
   Notices (datatoolkit-issues#243, #262, web #169): the "edits to workspace X
   were lost" notices are a list, one per workspace, dismissed one by one (a new
   failure on another workspace no longer replaces them); deleting or renaming a
   workspace removes its notice; a pending save failure follows a rename A -> A2
   and is never reported as lost; the startup error screen
   (`BootstrapGate.tsx`, extracted from `main.tsx`) keeps the name and reason of
   the skipped workspace across Retry. Note: two files must never differ only by
   case (macOS/Windows builds break); the project CHECK runs on Linux so check
   `git ls-files | tr A-Z a-z | sort | uniq -d` and the macOS/Windows CI jobs.
   Load feedback follow-ups (datatoolkit-issues#250, #239, web #170): Cancel
   shows a neutral "Load cancelled" notice (not an error); the in-flight GET
   dedup (`src/api/requestDedupe.ts`) is reference-counted per subscriber, so
   one caller cancelling does not abort a shared request for the others and the
   fetch is aborted only when every subscriber cancelled; the `loadMore` lock
   belongs to its generation (a stale page cannot reopen it); the Sources file
   row shows "Empty file" or "Not a readable table" with the engine's message,
   and such a file is not a ready Train X (workbench and alignment blocked until
   it is removed or replaced). Not done: a client-side request timeout.
   Dev proxy target (datatoolkit-issues#238, web #171): `DTK_API_PORT` (default
   8765, validated in `scripts/apiTarget.ts`, falls back on empty / non-numeric /
   out-of-range values) sets the port `npm run dev` proxies `/api` to; the
   production build and the e2e Vite config are unchanged. The README is short (since web #184, datatoolkit-issues#323, 59 lines) and
   links `docs/running.md` (quickstart #240, `DTK_AGENT_PACK` table #222),
   `docs/agent.md` (bridge, panel, packs), `docs/e2e.md`, `docs/behaviour.md`,
   `docs/development.md`; it cites FRONT-WEB.md / ARCHITECTURE.md of this hub.
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
   **CSV read engine** (web #182, datatoolkit-issues#321): Sources > Options of
   a CSV has an `engine` select (pandas default, duckdb). Choices come from
   `CsvSource.engine.enum` in a key schema (`GET /api/keys/dataset_overview/schema`),
   never hard-coded; hidden if fewer than two. `engine` is sent only when not
   pandas. No availability probe: duckdb is tried, and a `SourceError` naming
   the `large` extra shows a notice, resets to pandas and re-reads, keeping the
   other typed options. Measured (engine 43ea5ad, 600k rows / 93 MB): load 5.2 s /
   695 MB peak (pandas) vs 3.2 s / 893 MB (duckdb); Workbench profile 27.8 s /
   1566 MB vs 29.6 s / 1645 MB; parquet export 6.5 s / 1190 MB vs 6.1 s / 1561 MB.
   On 232 MB and 619 MB files both engines passed 2.5 GB server peak and were
   abandoned: duckdb does not lower peak memory in Studio because Sources
   preview and Workbench load a whole frame (engine-side, reported to the
   owner). The engine also writes `engine: "pandas"` into saved workspace
   sources.
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
   - **2026-10-14 (#196, #194)**: Sources target card has a "Keep target
     column last" toggle (`Workspace.label_last`, default off, undoable, part
     of the data identity); in the agent panel a paste over 4000 characters
     becomes a text chat attachment (`src/bench/agent/attachments/pasteAttach.ts`),
     kept as a workspace document only on request.
   - **Documents and agent memory (2026-10-06, #178, #179)**: a Documents
     section in the Sources card (upload by button or drag and drop, open /
     download, note, remove; PDF text when the engine has the `pdf` extra); a
     Memory view in the agent panel (list, edit, delete, clear). Notes,
     documents and memory travel in the workspace PUT (the Sources save keeps
     them) and each change is one Undo entry. The step picker lists every
     engine op, including the #162–#167 basics (`src/bench/stages.ts`).
   - **Epic #160 behaviours (2026-10-06)**:
     **Reorder columns** (#161): drag a grid header onto another (left / right
     half = before / after) → one `reorder_columns` step (target `both`); a
     drag right after folds into that last step, a drag back to the input order
     removes it; only at the latest version, no editor open, no preview pending;
     "removed" diff columns are neither draggable nor drop targets
     (`src/bench/grid/reorderDrag.ts`).
     **Step ids** (#153): `s` + 8 hex minted on create, kept by replace / editor
     apply / undo, missing ids filled on load with the engine's `s<position>`
     rule (`src/state/stepIds.ts`); agent `propose_steps` by id rebases
     (`staleReason` in `src/state/agentCommands.ts`, see `AGENT-BRIDGE.md`).
     **Notes** (#152): note button on step cards and the sources card
     (workspace note), column menu "Add / Edit note…" + header marker; popover
     textarea (max 4,000), Save / Delete / Cancel, Ctrl+Enter saves; one Undo
     entry per change (history snapshots steps + notes); column notes shown by
     the viewed version's names via `POST /workspace/column-notes`, stored
     under the origin key; notes travel in the PUT only, never in frame
     bodies; agent `set_note` uses the same path (toast + Undo).
     **Agent usage** (#151): panel foot shows session totals (uncached in ·
     cache write / read · out) and the last turn with its context size; a
     `compacted` event prints "Context compacted".
     **Export formats** (#156): Export panel checkboxes Parquet (default) /
     CSV / Notebook (.ipynb) / Python script (.py), sent as `formats`.
   - **Grid**: header = name, kind chip, mini histogram / top values, stacked quality bar (valid / mismatched / missing, from the column profile; mismatched = sentinel candidates only, approximate; web #174, datatoolkit-issues#310). Frames above 100,000 rows request a sampled profile (`sample_rows=100000`, engine #312) and the grid shows a "Sampled profile" notice (X of N rows, M not profiled); smaller frames keep the full profile (web #180, datatoolkit-issues#318). No DuckDB engine choice in the UI yet.
   - **expected_schema step** (Clean stage): dedicated rule editor for `columns` (type, required, nullable, unique, min/max, max missing %, allowed values; generic form cannot render the nested `$ref`/`anyOf`), violations table under the preview from the per-step `checks`, `on_violation=error` shows the engine 422 and disables Apply (web #179, datatoolkit-issues#316).
  
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
     Compare's stats come from the engine on the full frame
     (`column_distribution` for mean / median / std / min / max, `correlations`
     for r with the target; #75), never from the grid's loaded rows; the
     Correlation window sends no `columns` without a selection, so the engine's
     50-column cap and its "N more" note apply (#74). Chart sends nothing until
     its required fields are picked (X, plus Y for scatter / heatmaps; #80).
     **Analysis window shell** (MAT-235, generic, driven by the `Result`; all
     windows except Compare's native matrix and Chart): one compact chrome
     line (role · source, scope chip, Split by for Distribution, Parameters
     chip, "bound to…" truncated with tooltip — MAT-246) → `Result.headline`
     (1 line in short windows, 2 otherwise) → the selected figure filling the
     window (min 180 px; the default dock height fits 3 windows with full
     axes, no scroll) → view tabs (one per figure + Table; default = the
     `main: true` figure, else the first; choice kept per window in
     `toolViews`, in memory; the tabs fold into a select when they actually
     overflow, measured, datatoolkit-issues#73) + front-only display controls (sort / top-N,
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
   - **Grid view value facets** (web #177, datatoolkit-issues#276): the grid filter form lists a text column's `top_values` as tick boxes with counts; ticking builds an `isin` condition for the VIEW only (no step, data identity unchanged; "Make it a step" copies it); when `distinct` > `top_values` it says "Showing the N most frequent of M distinct values.". Numeric columns have no `top_values` and keep the typed input.
  - **Merge value groups** (web #178, datatoolkit-issues#277): each "Value groups" row in the column inspector has a Merge button opening a `replace_values` step (variant -> strip+lower canonical form) in the usual editor with live preview; nothing changes until Apply, undoable; disabled on a non-latest version. Variants come from `top_values` plus loaded rows.
  - **Step groups polish** (web #176, datatoolkit-issues#275): opening a step editor leaves group-select mode and clears the picks; `AppState.usedGroupIds` makes an ungrouped singleton take the engine's clash form (`g2-2`) instead of reusing `g2`; the failing-group state is axe-scanned.
  - **Steps file** (web #175, datatoolkit-issues#311): Export panel "Download steps" (checkbox per step) writes `{format: "dtk-steps", version: 1, steps: [{op, target, params, note?}]}` (no ids, no `align`); "Import steps…" validates the file at the boundary (`state/stepsFile.ts`: format/version, ≤500 steps, ≤1M chars, known op, params object) and appends via `APPLY_STEP_BATCH` (new ids, one undo entry). Missing columns fail at replay with the engine's error; fitted state is not carried (steps refit).
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
| `POST /workspace/rows` `{workspace, role, version, offset, limit, columns?, filter?, sort?}` | `workspace_rows` (MAT-127; optional `columns` filter MAT-152; view-only `filter` / `sort`, `total_unfiltered`, datatoolkit-issues#87) |
| `POST /workspace/profiles` `{workspace, role, version, columns?}` | `column_profiles` (optional `columns` filter MAT-152) |
| `POST /workspace/preview-step` `{workspace, step, role}` | `preview_step` |
| `POST /workspace/align` `{workspace}` | `align_report` |
| `PUT /uploads/{filename}` (raw body) → `{path}` | content-addressed under `$DTK_UPLOAD_DIR` (default `$DTK_HOME/uploads`); over `DTK_MAX_UPLOAD_MB` (engine, default 512) → 413 `UploadTooLargeError`, whose message Studio shows next to the upload button (Sources, Documents), as a `too large` chip on agent attachments (web #172) |
| `PUT/GET /ui/context` · `GET /ui/events?session=&token=` (SSE) · `POST /ui/ack` · `POST /ui/commands` `{type, …}` (`?session=&timeout=`) | UI bridge, not the contract (`dtk_engine/ui_bridge.py`, datatoolkit-issues#62; `AGENT-BRIDGE.md`) |
| `GET /ui/sessions` · `GET /ui/commands/schema` · `GET /ui/commands/{id}` | live Studio sessions, published Studio command schema (MCP tools are generated from it, #100), command status incl. `pending_review` (#95) |
| `GET /ui/agent?session=` · `POST /ui/agent/send` `{session, text, attachments?}` · `POST /ui/agent/cancel` · `POST /ui/agent/permission` `{session, id, allow}` | agent chat (events `event: agent` on `/ui/events`; wire format in the engine's `docs/agent-chat-protocol.md`) |
| `GET /ui/agent/options[?refresh=1]` · `POST /ui/agent/config` `{session, pack, model?}` | packs and models offered, per-session pack and model (#118) |
| `POST /ui/agent/attachments` `{session, path}` · `GET /ui/agent/attachments?session=` · `DELETE /ui/agent/attachments/{id}?session=` | read-only chat attachments, files under the upload dir only (#121) |
| `WS /ui/terminal?session=&pack=&model=&cols=&rows=&token=` | terminal packs, a CLI in a PTY (opt-in `dtk-api --terminal`; close codes 4401 / 4403 / 4404 / 4409 / 4422, #120) |
| `/mcp` (outside `/api`) | MCP streamable HTTP, same token / Origin / Host guard (#64) |

Errors: `{type, message}`; 404 for `UnknownKeyError`, `UnknownTransformError`,
`WorkspaceNotFoundError`; 422 for `KeyParamsError`, `SourceError`; 500 for
`WorkspaceCorruptError` (an unreadable workspace file, #206). A params
validation failure gives a concise `message` (`<loc>: <msg>; …`, never the
pydantic dump) plus `details: [{loc, msg, type}]` (MAT-142). CORS allows the
Vite dev origin (localhost / 127.0.0.1 :5173) plus `DTK_CORS_ORIGINS`
(comma-separated). `dtk-api --port 8765` runs uvicorn.

The front builds each key's params from `GET /keys/{id}/schema` (only the
properties the key declares, e.g. `test` / `target`), and dedupes identical
in-flight requests; one rows page + one profiles call per (steps, role,
version), workspace `PUT` only when its JSON changed (MAT-144).

**Agent UI bridge (datatoolkit-issues#62, #63).** The `/ui/*` routes need the
per-run token (`Authorization: Bearer`, or `?token=` for `EventSource` and the
terminal WebSocket, as browsers cannot set headers on either; engine
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
`src/state/stepOps.ts`). Chat v2 modules: `src/bench/agent/panel/` (chat: client,
protocol, transcript reducer, Markdown, pack / model picker),
`src/bench/agent/terminal/` (xterm panel, socket, protocol),
`src/bench/agent/attachments/` (upload via `PUT /uploads`, then register).
Semantics: `AGENT-BRIDGE.md` → "As built".

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
Unit (Vitest, `tests/**/*.test.ts`, plus colocated `src/**/*.test.ts` for the
agent terminal / attachments modules) for pure logic: selection, diff
colouring, dock grid layout (`src/state/dockLayout.ts`, MAT-234), schema →
editor fields, sources / alignment / workspace-manager logic, chart picker and
prefill, save gate. Coverage: `npm run coverage` (v8, no threshold; baseline
45 % lines on 2026-10-15, StepEditor / Grid / DockWindowBody / WorkspaceSidebar /
useBenchFrame at 0 %, e2e only). Gate (`fleet gate`): lint (ESLint, with
`sonarjs/cognitive-complexity` ≤ 25), typecheck, unit, `npx knip@6 --exclude
types` (knip.json enables the eslint, playwright, typescript, vite, vitest
plugins and `e2e/global-teardown.ts` explicitly, web #187, #326; 33 unused
exported types remain listed, not deleted); the same checks run in `.github/workflows/ci.yml` (datatoolkit-issues#3).
Placeholders (web #189, datatoolkit-issues#328): shared skeleton and spinner
primitives built on `--dtk-skeleton-*` and `--dtk-grid-row-h` /
`--dtk-grid-header-h` tokens; used for unloaded grid rows, profile, stats and
suggestions, with reserved dimensions (no layout shift, asserted in e2e),
shimmer off under reduced motion, status regions for assistive tech. Details:
`docs/placeholders.md` in the web repo. Windowed reads and the approximate-value
marker (datatoolkit#210) are not built yet.
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
(`waitForGridReady`), never a loading state. No fixed sleeps: ESLint
`playwright/no-wait-for-timeout` is an error on `e2e/**`. Accessibility:
`e2e/a11y.ts` runs axe (WCAG 2.0/2.1 A + AA) at the final state of 5 critical
journeys (Studio loaded, Sources with a file, workbench after steps,
save-failure banner, agent panel) and fails on any violation; 0 violations
and an empty allow-list on 2026-10-15 (datatoolkit-issues#241, web #154).
Full suite: ~10.5 min idle (12-17 min under load) on 1 worker; stabilised by
datatoolkit-issues#244 (web #155: 3 green runs in a row). Two Playwright
workers are not safe (one engine, one shared `DTK_HOME`). Project CHECK:
`bash scripts/fleet-check.sh` (web #159): `npm ci --prefer-offline`, lint,
typecheck, vitest, knip, then the full e2e on two free ports with
`DTK_ENGINE_DIR` (default `~/datatoolkit`); CHECK timeout 2400 s. The e2e takes
~13 min and is silent for ~30 s after "Uvicorn running" (Vite warm-up): that is
not a hang. The parkison step-groups spec builds its own workspace in the temp
`DTK_HOME` (web #183, datatoolkit-issues#322); it used to read the developer's
`~/.datatoolkit`.

## State (2026-10-06)
Epic datatoolkit-issues#160 (parkison agent dogfood, web #124–#128): drag a grid
header to reorder columns (one `reorder_columns` step, consecutive drags fold
into it, a drop on a removed diff column is ignored); stable step ids and
`propose_steps` rebase by id with a `stale: [{id, reason}]` ack
(`src/state/stepIds.ts`); agent usage counter splits uncached / cache write /
cache read / out and shows a "context compacted" line; notes on steps,
columns and the workspace (`src/bench/notes/`, icon + popover, undoable, agent
`set_note`); Export panel format picker (parquet / csv / ipynb / py).

## State (2026-10-04)
Chat v2 (web #107–#112): sanitised Markdown replies, collapsed tool chips, mode /
pack / model selector, opt-in terminal panel (xterm over `WS /api/ui/terminal`),
read-only chat attachments. Audit wave (engine #105–#111, web #113–#118): agent
path guard refuses `$DTK_HOME/agent`, UI token redacted in uvicorn logs, PTY
back-pressure, idle chat sessions reaped after 5 min, bounded API-pack history,
attachment detach (removed-while-uploading, shown and detachable chips), dev
server reads the token from the engine runtime file, throttled streaming
Markdown render, one agent HTTP plumbing. Earlier state below.

## State (2026-10-03)
Agent chat panel and 16 bridge commands merged (web #86–#100, see `AGENT-BRIDGE.md`); grid
view-only filter / sort, report-only analyses as dock windows, "By key column"
key picker with match report (#76, #81, #82). Earlier state below.

## State (2026-10-02)
Built and merged: engine datatoolkit #30–#88, web datatoolkit-web #1–#85
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
2026-10-02 (web 7024d47): 73 e2e tests, 360 unit; full run 72/73 under load,
the one failure a stale expectation fixed in #85 and re-run green (agent
bridge spec `issue63-agent-bridge`, dogfood specs for #73, #75, #80 added).
Deleting a workspace waits for the save already on the wire, then undoes it
(`src/state/workspaceSaveGate.ts`). Workbench open on 55 603 rows: ~1.5 s.
Known gaps: the export outputs list scrolls rather than showing all lines at
once.
`GET /workspaces/summaries` returns cached `train` / `test` `shape`
(MAT-204; content-addressed, no frame load when fresh).

**Review 22 follow-ups (2026-10-25).** Review polling no longer swallows errors:
`console.warn` then a "status unavailable" chip after 3 consecutive failures,
reset on success (web #185, #324, demo with a killed engine). `renderMarkdown`
has an XSS test, 29 script / `onerror` / `javascript:` payloads, no code change
(web #186, #325). Open, Matteo's call: the installers
(`scripts/datatoolkit.sh|.ps1`, `compose.yml`) run unpinned code from `main`.
