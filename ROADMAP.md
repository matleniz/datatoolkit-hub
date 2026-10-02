# Roadmap — dated journal (historical)

## 2026-09-24 — bootstrap
- Linear project `datatoolkit` (team MAT), fleet project `datatoolkit`
  (queue Linear, gate = ruff + pytest).
- Hub seeded: architecture/contract, stack allow-list, key cards, howtos.
- Skeleton built by a Sonnet worker (`dtk-skeleton`, PR #1, gate green, reviewed): engine + contract +
  `hello` key + generic Streamlit front + isolation barrier.

## 2026-09-24 — design: analysis / transform / pipeline
- Scope agreed with Matteo: analysis first, then transforms and reproducible
  pipelines; plug-and-play inputs (clean CSV with pandas now; malformed CSV,
  parquet via polars/duckdb, upload later). See `CAPABILITIES.md`.
- Engine split into sources → ops → keys; `SourceSpec` discriminated on `kind`;
  strict params; recursive generic front forms (`ARCHITECTURE.md`).
- Slice 1 merged (PR datatoolkit#2, MAT-8): csv source, `ops/profile`,
  `dataset_overview`, demo train/test CSVs, strict `KeyParams`, recursive
  front forms (`schema.py`). Gate: ruff + 42 tests. Review follow-ups:
  `semantic_type` heuristics, slow `sep="auto"` on big files.

- Hardening merged (PR datatoolkit#3, MAT-24): `group_id` type, fewer false
  `id_like`, `pct_numeric_parsable`, years not datetime, Windows paths, fast
  sniffing (X_train read 1.0 s → 0.46 s). Gate: 69 tests.

- Slice 2 merged (PR datatoolkit#4, MAT-25): `train_test_check` (`ops/compare.py`:
  schema, missing, ranges, categories, row / id overlap, severity-ranked
  issues); `hello` removed. Review fix before merge: row overlap now ignores
  ids / row counters, int vs float dtype = info. Gate: 91 tests. On the real
  X_train / X_test: `time_since_diagnosis` only in test, `age_at_diagnosis`
  5 % → 0 % missing, 23 identical feature rows across splits (different
  patients, sparse rows), no `patient_id` leak.

- 2026-09-25 — workspace design validated (engine-side JSON, step log =
  future pipeline, `kind="dataset"` source, tabs per semantic type, stat ops
  can fit on train). Three parallel workers: workspace, richer overview stats,
  train/test drift.

- 2026-09-25 — three slices merged: drift in `train_test_check` (PR #5,
  MAT-30: SMD, KS, PSI, % outside train p1–p99, TVD, overlaid histograms;
  p1–p99 info threshold raised 1 → 3 % after review), richer
  `dataset_overview` with tabs per semantic type (PR #6, MAT-29; Result
  `group` field), workspace + `dataset` source + label join + front top bar
  (PR #7, MAT-28). Gate: 168 tests on the combined merge. Real data: no drift;
  `age_at_diagnosis` test missing values imputed with a single value (56.3 on
  1 162 rows); labeled train 55 603 × 13, no row lost.

- 2026-09-25 — course backlog planned from Matteo's data collection /
  preprocessing course (`COURSE-MAP.md`). Decisions: split into two repos
  (engine alone installable, Streamlit apart), three doors to one `ops/`
  implementation (JSON contract, notebook `api`, sklearn `DtkTransformer`),
  fit/apply transform protocol; scikit-learn, pyarrow, openpyxl, sqlalchemy
  approved. Linear MAT-38 … MAT-47 in 4 batches: B0 split + foundations
  (sequential, opus) → B1 sources + quality keys ∥ B2 transform ops → B3
  front panel + advisor/export.
- 2026-09-25 — repo split merged (PR datatoolkit#8, MAT-38): engine flattened
  at the root of `datatoolkit` (147 tests), front in new private repo
  `datatoolkit-streamlit` (21 tests, own fleet project, same hub and Linear
  queue), front pinned to engine `main`.
- 2026-09-25 — foundations merged (PR datatoolkit#9, MAT-39): fit/apply
  transform protocol + replay, `list_transforms` / `transform_schema`,
  notebook `api` + `Result._repr_html_`, `DtkTransformer` /
  `workspace_pipeline`, deps sklearn / pyarrow / openpyxl / sqlalchemy.
  Gate: 172 tests. Batch 1 + 2 dispatched (6 workers, MAT-40 … MAT-45).
- 2026-09-25 — batches 1 + 2 merged (6 parallel workers, PRs datatoolkit#10–15):
  sources parquet / excel / json / sql + csv options + `file_inspect` (MAT-40),
  `duplicates` + `inconsistencies` (MAT-41), `missing_values` + `outliers`
  (MAT-42), 26 transform ops — cleaning (MAT-43), impute / encode / scale
  (MAT-44, KNN / iterative state = train matrix), features (MAT-45). Conflicts
  only on `keys/__init__.py` / `api.py` (unions). Gate: 292 tests. Catalog:
  `TRANSFORMS.md`. B3 dispatched (MAT-46 front, MAT-47 advisor / export).
- 2026-09-25 — front Transforms panel merged (datatoolkit-streamlit#1, MAT-46,
  26 front tests). Fleet bug: an inherited `FLEET_CONF` overrides `--project`
  (MAT-53); front dispatch uses `env -u FLEET_CONF -u FLEET_PROJECT`.
- 2026-09-25 — advisor + export merged (PR datatoolkit#16, MAT-47): key
  `preprocessing_advisor` (per-column plan as workspace steps, per model
  family), `export_workspace` (parquet + provenance manifest, big states in
  side files). Review fix before merge: overwrite only deletes validated paths
  of a `dtk_engine` manifest. Gate: 329 tests. **The course backlog
  (MAT-38 … MAT-47) is complete.**
- 2026-09-25 — follow-ups merged: `align_to_train` complete (PR #17, MAT-58:
  shift_mean / shift_median / standardize / robust / quantile, per-group,
  small-frame guard), `preview_workspace` in memory (PR #18 + front #2,
  MAT-55), feature selection from course 11 (PR #19, MAT-56: key +
  5 selection ops incl. PCA, `needs_target` for supervised ops in sklearn).
  Gate: engine 381 tests, front 27.

- 2026-09-26 — whole Linear backlog processed (coordinator + fleet workers, no
  human in the loop). Engine PRs datatoolkit#20–29: json/excel fixes (E2),
  robust csv + CSV-aware `file_inspect` with `load_spec` (E1), nested / binary
  columns + mixed date formats + typed feature_selection errors (E3), step
  validation at save / error taxonomy / single replay loop (E4), comparison
  keys `column_distribution`, `target_analysis`, `correlations` + column-selector
  convention + `source_columns` (C1, MAT-94), ops refactor into `advisor/` /
  `compare/` packages (E5), contract `needs_target` flags + `Table.kind` steps
  + engine doc honesty (E6), dogfood fixes (MAT-114 encoding on big samples,
  MAT-115 excel header), transform column-selector hints (MAT-119). Front PRs
  datatoolkit-streamlit#3–6: export / apply steps / needs_target / clean errors,
  column selectors + dataset input for any format with upload, workspace target
  prefill, Transforms-panel selectors. Gate: engine 523 tests, front 71.
  End-to-end check on a personal dataset (cp1252 `;` / decimal-comma csv +
  xlsx test, y in X): workspace → 12 keys → advisor steps applied → re-analyse
  → add / undo step → export, no error.

## 2026-09-27 — web front "Studio" (MAT-126)
Streamlit felt like a form catalog. Matteo validated a from-scratch design,
prototyped on Claude Design (Sources → train/test alignment → workbench:
pipeline bar, Excel-like grid, right-click, manual step editor showing what is
learned on train, variables + formulas, suggestions as a tab, dockable tools).
Approved: React + TS + Vite, FastAPI + uvicorn, Vitest, Playwright; new repo
`datatoolkit-web`; engine gains grid contract (MAT-127), `formula` op
(MAT-128), merges + variables (MAT-129), HTTP API (MAT-130). Spec: `FRONT-WEB.md`.

## 2026-09-28 — Studio built and e2e-validated
Web repo merged #1–#10 (scaffold, sources + alignment, workbench, panels +
dock, e2e suite), three screenshot-review rounds (MAT-139, MAT-142) and a
real-data dogfood (MAT-144: duplicate API calls cut, workbench open 21 s →
1.5 s on Parkinson). Engine #34: no sentinel alerts on identifier columns,
concise validation errors with `details`. Fleet gap found overnight (a worker
blocked 8 h on a foreground dev server) → MAT-140 (Agent Fleet).

## 2026-09-30 — Docker, one-command install (MAT-255, MAT-256)
Matteo asked for a Docker setup runnable anywhere in one line; chose GHCR images
published by GitHub Actions (Docker, Actions, GHCR added to `STACK.md`). Engine
datatoolkit #64 (image, entrypoint fixing root-owned bind mounts, PR smoke job);
web datatoolkit-web #49 (nginx image, `compose.yml`, full-stack smoke) and #50
(wait for nginx: #49's smoke raced on main). No local Docker daemon, so all
checks run in CI. See `HOWTO/run-with-docker.md`.

## 2026-10-01 — cleanup, Streamlit archived, code-quality priority
Matteo set code quality as the priority before any new feature. Cleanup: web
#50 merged (MAT-256 done); 27 merged remote branches deleted and
`delete_branch_on_merge` enabled on both repos; orphan worktree and 552 e2e
temp dirs (~420 MB) removed (cause filed as MAT-257). The unmerged
`qa-exercises` QA report (2026-09-28, findings closed as MAT-160) moved to
`reports/qa-exercises.md`. The Streamlit front is archived (GitHub read-only,
clone in `~/Archives`, fleet project retired); the hub now describes Studio as
the only front. MAT-53 (inherited `FLEET_CONF`) was fixed on 2026-09-26: a
plain `fleet --project datatoolkit-web …` is enough.

Filed: engine and web code-quality audits (MAT-258, MAT-259 — tool-driven,
coordinator audits, then one lead worker per repo with scoped sonnet
sub-workers; MAT-218 / MAT-219 folded in), then impute by formula (MAT-260),
per-entity regression / interpolation imputation (MAT-261, Parkinson case) and
the in-Studio agent spike (MAT-262).

## 2026-10-01 — code quality, wave 1 (MAT-258, MAT-259)
Coordinator lesson: audits are delegated. One opus lead per repo audits with
recognized tools (engine: ruff extended, radon, vulture, jscpd, import-linter,
deptry, pyinstrument; web: knip, dependency-cruiser, jscpd, sonarjs, bundle
stats), files sub-issues and drives sonnet sub-workers (`--scope`
+ `--checks scope:blocking`); the coordinator reviews and merges.
Engine #65–#68: one cache module `dtk_engine/cache.py` with memoized column
facts (warm Parkinson `column_profiles` 1.6 s → 0.04 s, `preprocessing_advisor`
1.05 s → 0.03 s; MAT-218 closed), `dataset` source moved to `workspace/`,
formula engine = one function registry + one AST pass (MI B → A), shared
`SourceParams`, dead code removed; 98/98 contract outputs identical. Web
#52–#56: shared field controls + keyed async hook (MAT-219 closed), pure
reducer, Sources state object, Inspector / AnalysisResultView split, dead
`mockClient` (−1.3k lines net). Verified on both merged mains: 921 engine
tests, 245 web unit tests, build, full e2e 61/61. Blockers found: the Linear
workspace hit its free-issue limit; haiku cannot run headless.

## 2026-10-01 — code quality wave 2; queue moved from Linear to GitHub
Wave 2 merged and verified on both mains (engine 922 tests, web 253 unit,
build, full e2e 61/61). Engine #69–#74 (MAT-258): `create_app` split into
APIRouters, `find_issues` / advisor / key results / column scans / chart and
csv dispatch as rule tables; ruff complexity hits 33 → 0, no radon block worse
than C except `file_inspect._text_facts`; contract and OpenAPI byte-identical.
Web #57–#62 (MAT-259): e2e writes `docs/screenshots` only with
`DTK_E2E_SCREENSHOTS=1`, `useDebounced`, DockWindowBody run state, op knowledge
read from the transform schema, WorkbenchData split into hooks; functions above
cognitive complexity 15: 36 → 22.

Linear hit its free-plan issue limit. Both queues moved to **private GitHub
issue repos** with a board each: `matleniz/datatoolkit-issues` (Project #1) and
`matleniz/agent-fleet-issues` (Project #2); the README of each repo is the
contract (roles, lifecycle, labels). `fleet issue new|start|review|block|done|
bootstrap` drives them (agent-fleet #22); fleet also routes Claude models to the
claude pack and stops false out-of-scope reports (#21, #23). The 16 open
datatoolkit issues were migrated (#7–#22; Linear copies canceled with a link);
new epics: #1 code-quality follow-ups (gate decision #3 for Matteo), #2
imputation beyond fixed strategies (#7, #8). Everything shared is in English.

## 2026-10-01 — Matteo's decisions, docs refresh, next wave
Decisions recorded on the issues: light gate #3 adopted as recommended
(eslint-plugin-sonarjs added to STACK.md); #8 group-aware impute go; #15
dismiss suggestions go; #16 undo / redo only, low priority; #18 `.xlsx` only,
clear error, no new dependency; #17 closed (not planned). #9 (in-Studio agent)
is **paused** (see Next). Docs refresh verified against the code (#28):
engine #78 and web #67 READMEs merged, hub proposals #29–#36 and #38–#43
applied. Merged: light gate #3 (engine #77: ruff C90 / PLR / SIM / PERF / B +
`tests/test_layers.py`; web #68: knip, sonarjs ≤ 25, `ci.yml`; knip added to
the web `GATE_CMDS`), #18 (engine #79). Then #7 impute formula (engine #80),
#8 group-aware fills (engine #82, re-opened from #81 after its stacked base
merged), #15 dismiss suggestions (web #69), #16 undo / redo (web #70); doc
proposals #44–#47 applied. Engine main: ruff clean, 979 tests; web main: 284
unit, knip clean.

## Next (see matleniz/datatoolkit-issues)
- #37 needs Matteo (drop or list the unused `jsdom` devDependency).
- Done after Matteo's second go (#37, #10, #14): web #71 (jsdom dropped),
  #72 (#48: `x-dtk-when` / `x-dtk-semantic` / group functions in Studio; fixes
  the flow3 regression from the new impute params), #73 (#10: edit an applied
  step, undoable), #74 (#14: e2e Vite pre-bundles deps). Then #12 (web #75:
  formula error under Expr), #11 (web #76: saved charts on
  `Workspace.charts`), #13 (web #77: delete waits out the in-flight save,
  e2e readiness signals). Final check on main (engine 4dcca8b, web 2a31aea):
  e2e 65/65 on a cold cache (5.4 min), 308 unit, engine 979 tests.
- agent-fleet issues filed: #8 (`fleet wait` / `ls` report busy e2e workers as
  stalled / blocked on vite), #9 (`test-server-detect.sh` leaks http.server
  processes).
- Queue empty except the paused #9.
- #9 in-Studio agent: paused 2026-10-01, resumed 2026-10-02 for design; see
  the 2026-10-02 entry below and `AGENT-BRIDGE.md`.

## 2026-10-02 — share it: one-line launcher, Docker or uv
Docker launchers `scripts/datatoolkit.sh` / `.ps1` (web #78, #53), then the
no-Docker uv mode (web #79, #55: `dtk-studio` serves Studio + engine in one
process, Studio build as release asset `studio-latest`); it surfaced that the
engine did not import on Windows (`fcntl`), fixed with `msvcrt` locking
(engine #83, #57). CI green on ubuntu / macOS / Windows; the published uv
one-liner checked by hand on a fresh data home. Recipe:
`HOWTO/share-and-run.md`. Queue: only the paused #9.

## 2026-10-02 — next wave: join preview, csv_robust, dogfood, agent bridge design
Matteo picked `label_join_preview` (#58, engine key only, generic rendering),
the rest of `csv_robust` (#59: junk header lines, mixed separators), a Studio
dogfood on the real Parkinson data (#60, find only) and the #9 design. The
design spike produced `AGENT-BRIDGE.md` (doc proposal #61, approved with every
recommendation): an MCP server over the contract in an engine extra, a UI
bridge (UI context + SSE commands + ack, per-run token), Studio as the single
writer of agent edits (undoable), a path guard confined to `$DTK_HOME`, packs
(Claude Code / gemini / opencode external first, then an in-Studio chat panel
on the Claude Agent SDK). `mcp` and `claude-agent-sdk` approved in
`STACK.md`. Sub-issues: phase 1 #62 / #63, phase 2 #64–#66, phase 3 #67.
