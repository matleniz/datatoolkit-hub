# Repo context — datatoolkit-hub

## What this is
Documentation hub for **datatoolkit**, Matteo's personal DS/ML keyring: a
pure-Python analysis engine behind a JSON contract, with swappable visual fronts
(Studio, a React web app, today). Source of truth for the architecture. Navigate via `INDEX.md`.

## Where the code lives (not here)
- `~/datatoolkit` (github matleniz/datatoolkit) — the **engine** `dtk_engine`,
  a standalone Python package (notebook, scripts, any front). No front code.
- `~/datatoolkit-web` (github matleniz/datatoolkit-web, fleet project
  `datatoolkit-web`) — the **active** front, "Studio" (React + TypeScript,
  see `FRONT-WEB.md`), talks only to the engine's HTTP API. All current front
  work happens here.
- The first front (Streamlit) was dropped for Studio on 2026-09-27 and its
  repo archived on 2026-10-01 (`~/Archives/datatoolkit-streamlit`, GitHub
  read-only). Not a dispatch target.
All code changes happen in the engine or web repo.

## Architecture in brief
`dtk_engine` (keys = typed params → JSON `Result`) → contract (`list_keys`,
`key_schema`, `run_key`) → HTTP API (`dtk-api`) → generic front. Fronts never import
the engine; adding a key needs zero front code. One implementation in `ops/`,
three doors: JSON contract, notebook `dtk_engine.api`, sklearn `DtkTransformer`.
Details: `ARCHITECTURE.md`.

## Hard project rules
- **Only known tools.** New dependency → `STACK.md` + Matteo's approval first.
- Built key by key, on demand. Do not add keys or features nobody asked for.
- Everything shared is in English: issues, comments, PRs, commits, reviews.

## Your fleet — dispatching workers
You are the COORDINATOR: you write the docs and dispatch code work; you do not
edit the code yourself.

HARD RULE — every agent launch goes through `fleet` (`fleet w`, `fleet
dispatch`, bare `fleet`), never an agent CLI by hand: the launch carries the
posture (permission mode, read-only-hub barrier, MCP profile, resource guard).

- `fleet-queue` — where issues go: **`matleniz/datatoolkit-issues`** (private,
  board = GitHub Project #1). Conventions (roles, lifecycle, labels, issue
  body) live in that repo's README; read it before filing.
- `fleet -a claude dispatch --model opus <name> "<task>"` — a lead worker (one
  per repo / stream): it audits or designs, files sub-issues and drives
  sub-workers; sub-workers run `--model sonnet` (haiku cannot run headless).
  Trivial, easily checked tasks may go to `-a antigravity`. Always pass `-a`.
- You do not audit, profile or read code in depth yourself: brief a lead.
- `fleet ls` / `fleet status` / `fleet prune` / `fleet chats [<worker>]`.
- Several disjoint streams → `dispatch-work` skill (partition by file ownership).

## Issue queue (since 2026-10-01; Linear before, ids `MAT-<n>`, read-only)
- One issue = one deliverable; epics are `type:epic` with GitHub sub-issues.
- Labels: one `type:*`, one `priority:*`, `area:engine|web|hub`; `status:*`
  and the board column move together; `needs:matteo` = a decision only Matteo
  can take (body has a `### Decision needed` section with a recommendation).
- A worker starts an issue (status:in-progress + "Started by <name>"), opens
  one PR whose body says `Closes matleniz/datatoolkit-issues#<n>`, and never
  merges. The coordinator reviews, merges, updates the hub at merge time.
- Doc drift found by a worker → `type:doc-proposal` issue (skill
  `propose-doc-change`), never a hub edit.

## Conventions
- Verify facts against the code before asserting. Do not guess names/flags.
- Docs have one writer (the coordinator). Workers are read-only here and propose
  changes via `propose-doc-change`.
- Navigate cheaply: INDEX first, one file, grep sections.
- Drift is merge-driven: flag it, fix at a checkpoint, never rewrite the hub.
