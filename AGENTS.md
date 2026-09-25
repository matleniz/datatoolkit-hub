# Repo context — datatoolkit-hub

## What this is
Documentation hub for **datatoolkit**, Matteo's personal DS/ML keyring: a
pure-Python analysis engine behind a JSON contract, with swappable visual fronts
(Streamlit first). Source of truth for the architecture. Navigate via `INDEX.md`.

## Where the code lives (not here)
Two repos (split decided 2026-09-25, MAT-38):
- `~/datatoolkit` (github matleniz/datatoolkit) — the **engine** `dtk_engine`,
  a standalone Python package (notebook, scripts, any front). No front code.
- `~/datatoolkit-streamlit` (github matleniz/datatoolkit-streamlit) — the
  Streamlit front, depends on the engine via git.
All code changes happen there.

## Architecture in brief
`dtk_engine` (keys = typed params → JSON `Result`) → contract (`list_keys`,
`key_schema`, `run_key`) → `EngineClient` → generic front. Fronts never import
the engine; adding a key needs zero front code. One implementation in `ops/`,
three doors: JSON contract, notebook `dtk_engine.api`, sklearn `DtkTransformer`.
Details: `ARCHITECTURE.md`.

## Hard project rules
- **Only known tools.** New dependency → `STACK.md` + Matteo's approval first.
- Built key by key, on demand. Do not add keys or features nobody asked for.
- Linear (team MAT, project datatoolkit) is English-only.

## Your fleet — dispatching workers
You are the COORDINATOR: you write the docs and dispatch code work; you do not
edit the code yourself.

HARD RULE — every agent launch goes through `fleet` (`fleet w`, `fleet
dispatch`, bare `fleet`), never an agent CLI by hand: the launch carries the
posture (permission mode, read-only-hub barrier, MCP profile, resource guard).

- `fleet-queue` — where issues go (Linear here).
- `fleet dispatch [--model M] <name> "<task>"` — headless worker. One coherent
  change = one worker; mechanical work → `--model sonnet` with a precise brief.
- `fleet ls` / `fleet status` / `fleet prune` / `fleet chats [<worker>]`.
- Several disjoint streams → `dispatch-work` skill (partition by file ownership).

## Conventions
- Verify facts against the code before asserting. Do not guess names/flags.
- Docs have one writer (the coordinator). Workers are read-only here and propose
  changes via `propose-doc-change`.
- Navigate cheaply: INDEX first, one file, grep sections.
- Drift is merge-driven: flag it, fix at a checkpoint, never rewrite the hub.
