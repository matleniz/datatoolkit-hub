# First-user E2E — Studio, 2026-10-15 (night run)

A sonnet worker started from zero and followed only `README.md` (dev "Run"
path: engine `dtk-api` + `npm run dev`). It drove Studio headless with the
repo's Playwright, using a non-demo 200-row CSV (mixed types, missing values).
The Docker/uv one-liner (`curl … | sh`) was **not** run (no external network
in the sandbox).

## Journey (summary)

1. README "Run": works if `~/datatoolkit` is already cloned and synced with
   uv. No clone command, no Node prerequisite (→ #240).
2. Open Studio, create a workspace, upload `customers.csv`.
3. A single file is guessed "Merge" with 3 red errors (→ #235). Set by hand:
   Train X, target `churned` → 200×8.
4. Reload before "Open workbench": the Sources setup is gone, workspace
   empty on disk, upload orphaned (→ #234).
5. Open workbench, `+ Step` → Impute on `age` → Apply: step in the DAG and in
   the workspace JSON on disk.
6. Reload: the step survives; Studio reopens on Sources, not on the Workbench.
7. Corrupt workspace (JSON truncated to 50 %): silent fallback to `churn`,
   entry greyed with a "corrupt" badge, reason only in a tooltip (→ #236,
   related #220).
8. Bad uploads (empty, binary, ragged CSV): no per-file warning (→ #239).
9. Engine stopped: banner "Engine unavailable: HTTP 500 … Start dtk-api on
   port 8765", but the list says "No workspaces yet." and "+ New workspace"
   stays enabled (→ #237, related #219).
10. `dtk-api` on another port: the Vite proxy target is fixed at 8765 (→ #238).

Minor UX (noted in #240): reload reopens Sources rather than the Workbench;
two "Apply" buttons for the same selector in the step editor.

## Same night, read-only code review

#218 autosave failures swallowed (high), #219 bootstrap error screen,
#220 remembered workspace GET failure blocks startup, #221 no error boundary
/ no flush on `pagehide`, #222 README agent pack values inconsistent.
