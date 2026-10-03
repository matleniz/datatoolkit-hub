# Agent bridge — an agent in Studio, plugged per pack (design, datatoolkit-issues#9)

Status: **approved** by Matteo on 2026-10-02; phases 1, 2 and the first part of
3 built by 2026-10-03 (see "As built"); in-Studio chat on the `agent-sdk` pack,
external CLIs through `dtk-mcp config`.
Code: engine `~/datatoolkit`, front `~/datatoolkit-web`.

## Goals

1. An agent **sees what the user sees** in Studio: workspace, role (Train /
   Test), viewed version, selected columns / row / cell, open dock windows and
   their params, the step being edited.
2. The agent **acts through the engine contract only**: runs analysis keys,
   proposes / adds / edits / removes workspace steps, opens a window, selects
   columns, moves the viewed version — and Studio **refreshes live** through
   the existing refresh-identity contract (MAT-175), nothing new to refresh.
3. **Agent-agnostic.** One base (MCP server + UI-context protocol), the agent
   plugged per **pack** (Claude Code / Claude Agent SDK, a plain chat backend,
   opencode, gemini…), the way agent-fleet packs plug CLIs. The panel form
   (chat or terminal) follows from the pack.
4. Usable **without** the in-Studio panel: any MCP-capable agent the user
   already runs (Claude Code, gemini, opencode in a terminal next to the
   browser) plugs into the same server.

## Non-goals

- No shell, no generic file / network tool, no code execution for the agent.
  Data work goes through keys and transform ops; a missing capability is a new
  key or op (`HOWTO/add-a-key.md`, `HOWTO/add-a-transform.md`), never a tool
  that escapes the contract.
- No agent inside the engine core: `dtk_engine` (ops, keys, contract) stays
  agent-free; the bridge is an optional layer like the HTTP API.
- No multi-user / remote hosting: local, one user, same trust zone as
  `dtk-api` (bound to 127.0.0.1) or `dtk-studio`.
- No agent memory, RAG or auto-pilot; the user drives, the agent assists.
- Not in the Docker image in the first phases (the uv / dev paths only).
- No bundled API key, no billing of our own.

## Facts this design rests on (verified 2026-10-02)

- The contract (`dtk_engine/contract.py`) is plain JSON: `list_keys`,
  `key_schema`, `run_key`, `list_transforms`, `transform_schema`, workspace
  store calls, `preview_workspace`, `workspace_rows`, `column_profiles`,
  `preview_step`, `align_report`, `export_workspace`, `source_columns`. The
  HTTP API exposes it 1:1 (`dtk_engine/http.py`, `FRONT-WEB.md`).
- **Studio holds the live workspace** in its reducer (`src/state/reducer.ts`,
  `AppState.workspace`) and autosaves it with a chained, debounced `PUT`
  (`src/state/workspaceSaveGate.ts`). The engine store is last-write-wins: no
  revision, no ETag. There is **no engine → front push** today (no SSE /
  WebSocket / polling in `src/`).
- View state lives only in the front: `viewVersion` (null = latest),
  `selection {columns, row, cell{rid,col}, multi}`, `dock.tools` (`ToolId`:
  compare, corr, dist, missing, outliers, target, drift, feature_selection,
  chart), `toolParams`, `editor`, `screen`, `role`.
- Step edits already have reducer actions with undo: `ADD_STEP`,
  `REPLACE_STEP`, `REMOVE_STEP`, `UNDO_STEPS` / `REDO_STEPS`. `ADD_STEP`
  appends and jumps to the latest version.
- The data identity (`src/bench/dataIdentity.ts`) = workspace + role +
  effective version + FNV-1a hash of sources / label / merges /
  `steps[0:version]`; every data consumer keys on it.
- The contract is **not** path-confined: a `csv` / `parquet` / `excel` /
  `json` source reads any local path; a `sql` source reads the env var named
  by `url_env`; `export_workspace` writes to any `out_dir`. The formula op is
  a whitelisted `ast` walk, never `eval` (`ops/transforms/formula.py`).
- `dtk-api` binds `127.0.0.1` by default, no auth; `dtk-studio`
  (`launcher/src/dtk_studio/app.py`) mounts the Studio build on the same app,
  same origin `/api`.

## Architecture

```
                      ┌──────────── engine process (dtk-api / dtk-studio) ────────────┐
 agent (pack) ──MCP──▶│ /mcp  dtk_engine/agent/ (extra "agent")                         │
  claude / gemini /   │   tools = contract, scoped + guarded ──▶ dtk_engine.contract    │
  opencode / SDK /    │   ui tools ──▶ UI bridge (in memory: context + command relay)   │
  plain chat          │ /api/ui/context  ◀── PUT  ─────┐                                │
                      │ /api/ui/events   ──  SSE ───┐  │                                │
                      │ /api/ui/ack      ◀── POST ─┐│  │                                │
                      └────────────────────────────┼┼──┼────────────────────────────────┘
                                                   ││  │
                      Studio (browser) ────────────┴┴──┘  reducer = single writer of the workspace
```

### 1. MCP server over the contract

- New optional layer `dtk_engine/agent/` (extra `agent`), allowed imports:
  `dtk_engine.contract` + the MCP library + the UI bridge — same rule as
  `http.py`, enforced by `tests/test_layers.py`.
- Transports: **streamable HTTP mounted at `/mcp`** on the existing FastAPI
  app (one process with the store and the UI bridge, so an agent call can
  reach the open Studio), plus a **stdio** entry `dtk-mcp` for agents that
  only speak stdio (it proxies to the running `/mcp`, or runs contract-only
  without UI tools when no Studio is up).
- Tools (names final at implementation; each one = one contract call):

| Tool | Contract / bridge | Notes |
|---|---|---|
| `list_keys`, `key_schema`, `run_key` | same | `source` defaults to the viewed identity (`{kind: dataset, workspace, role, version}`); other sources go through the path guard |
| `list_transforms`, `transform_schema` | same | |
| `list_workspaces`, `get_workspace` | `list_workspace_summaries`, `get_workspace` | read-only |
| `get_rows`, `get_profiles` | `workspace_rows`, `column_profiles` | rows capped (default 50, max 500), columns filter |
| `preview_step`, `align_report`, `source_columns` | same | dry runs, no write |
| `get_ui_context` | UI bridge | also published as MCP resource `studio://context` |
| `propose_steps` | UI bridge → Studio | add / replace / remove steps; Studio applies through the reducer (undoable) and acks |
| `open_window`, `select_columns`, `set_view` | UI bridge → Studio | `ToolId` + params, columns, role / version |

  Not exposed: `save_workspace`, `delete_workspace`, `rename_workspace`,
  `duplicate_workspace`, uploads; `export_workspace` only later and only into
  `$DTK_HOME/exports/` (decision in the security sub-issue).
- Tool descriptions and input schemas come from the contract itself
  (`key_schema`, `transform_schema`), so a new key or op is an agent tool
  argument with no bridge change.
- Errors map like the HTTP API: `{type, message, details}`
  (`KeyParamsError` concise message, MAT-142), returned as tool errors the
  agent can read and retry.

### 2. UI-context protocol

Studio → engine, `PUT /api/ui/context` (debounced ~300 ms, on change):

```json
{
  "session": "<random id per Studio tab>",
  "screen": "bench",
  "workspace": "parkinson",
  "role": "train",
  "version": 4,            // effective version (viewVersion resolved)
  "latest": 6,             // steps.length
  "identity": "parkinson|train|v4|1a2b3c4d",   // DataIdentity.key
  "selection": {"columns": ["age"], "row": null, "cell": {"rid": 12, "col": "age"}},
  "windows": [{"tool": "dist", "params": {"column": "age", "by": "status"}}],
  "editor": {"op": "impute", "index": null, "params": {...}} // or null
}
```

The engine keeps the **latest context per session** in memory only (never on
disk, never in the workspace JSON). `get_ui_context` returns the most recent
session's; a tool may take `session` explicitly. Every agent read that
defaults to "what the user sees" uses `workspace` + `role` + `version` from
the context and echoes back the `identity` it answered for, so an answer is
always tied to one frame.

Engine → Studio, `GET /api/ui/events` (Server-Sent Events, browser
`EventSource`, `StreamingResponse` — no new dependency):

```
event: command
data: {"id": "c17", "type": "propose_steps", "workspace": "parkinson",
       "base_identity": "...", "ops": [{"add": {...step}}, {"replace": {"index": 2, "step": {...}}}]}
event: command
data: {"id": "c18", "type": "open_window", "tool": "dist", "params": {"column": "age"}}
```

Studio → engine, `POST /api/ui/ack {id, ok, error?, identity?}`: the MCP tool
call waits (timeout ~30 s) for the ack and returns its result — the new
identity on success, the reducer / engine error otherwise, `no_studio` when no
session is connected.

### 3. Writes and live refresh: Studio is the single writer

Agent step edits do **not** write the engine store. They are relayed to
Studio, which dispatches `ADD_STEP` / `REPLACE_STEP` / `REMOVE_STEP` (one undo
entry per command), autosaves through the existing save gate, and the new
workspace changes the data identity → grid, profiles, inspector, windows,
suggestions refresh by MAT-175 with no new refresh code. Why: the store is
last-write-wins, so a second writer would be silently overwritten by the next
autosave; Studio already owns undo, ordering (`orderSteps`) and the
"await the PUT before `run_key`" rule. `base_identity` lets Studio refuse a
command computed on a frame the user has since changed (ack `stale`).

Options for confirm: **apply then undo** (default: applied at once, a toast
"Agent added step X — Undo") or **review first** (Studio shows the proposal in
the step editor / a diff via `preview_step`, user clicks Apply). Destructive
ops (remove a step, drop rows / columns) always go through review.

Later (only if a headless agent must edit without Studio open): engine-side
writes with a workspace `revision` (`If-Match` on `PUT`) and a
`workspace_changed` SSE event that Studio reloads. Out of the first phases.

### 4. Packs: the agent plugged per adapter

A pack is a small descriptor + launcher, modelled on agent-fleet
`packs/<name>/pack.sh` (`pack_launch`, `pack_doctor`, `pack_install`…):

| Field | Meaning |
|---|---|
| `id` | `claude-code`, `agent-sdk`, `chat`, `opencode`, `gemini`, `stub` |
| `panel` | `chat` (structured events) or `terminal` (PTY) or `external` (none) |
| `detect()` | CLI on PATH / key env var present → shown as available |
| `mcp_config(url, token)` | the pack's MCP config (Claude Code `--strict-mcp-config --mcp-config <file>`; gemini `settings.json` `mcpServers` + `mcp.allowed`; opencode `opencode.json` `mcp`) with **only** the `dtk` server |
| `tool_policy()` | how the pack's own built-in tools are turned off (Claude Code / Agent SDK allowed-tools = `mcp__dtk__*`; gemini / opencode tool exclusion) — flags to re-verify per CLI version at implementation |
| `launch(...)` | start the agent (chat adapter loop, or PTY process) |
| `auth` / `cost` | where credentials come from, who pays (below) |

Panel follows from the pack:

- **external** (phase 2, every MCP-capable CLI): no panel; `dtk-mcp config
  <pack>` prints the config snippet; the user runs their own CLI. Their CLI's
  other tools are their business, outside our boundary.
- **chat** (`agent-sdk`, `chat`, opencode via `opencode serve`): Studio panel
  speaks one small event protocol over the same SSE / POST pair —
  `user_message`, `assistant_delta`, `tool_call`, `tool_result`,
  `permission_request` / `permission_reply`, `done`, `error`, `usage`. The
  adapter translates its agent's stream into it. DTK owns the tool list, so
  "contract tools only" is guaranteed.
- **terminal** (`claude-code`, `gemini`, opencode TUI): xterm panel over a
  WebSocket to a PTY running the CLI directly (no shell in between), with the
  pack's MCP config and tool policy. The CLI's own built-in tools can only be
  disabled as far as that CLI allows — weaker guarantee, so terminal packs
  are opt-in and labelled as such.
- **stub**: scripted tool calls, no network, for unit / e2e tests (same idea as
  agent-fleet `packs/stub`).

### 5. Security

- **Contract-scoped tools only**: the MCP server exposes the table above and
  nothing else; no shell, no file read / write, no network, no `eval`.
- **Path guard** in the bridge (not in the contract, so notebooks keep full
  access): every source path the agent passes must resolve (after
  `realpath`, symlinks followed) under `$DTK_HOME` (uploads included) or be a
  path the active workspace already references; `sql` sources refused (the
  `url_env` indirection would let the agent pick any env var); `export` only
  into `$DTK_HOME/exports/<workspace>` if exposed at all.
- **Writes are proposals**: relayed to Studio, undoable, destructive ones
  reviewed; no workspace delete / rename / overwrite tool.
- **Prompt injection from data**: cell values and column names reach the model
  as data. Mitigation by construction (no dangerous tool exists, every write
  is visible and undoable), plus tool results wrap data in a clearly delimited
  field.
- **Data egress**: with a cloud model, rows and profiles the agent reads are
  sent to that provider. Row caps by default; the panel states the provider;
  a local model through the `chat` pack keeps data on the machine.
- **Local endpoint hardening**: `/mcp` and `/api/ui/*` require a per-run
  random bearer token (generated at start, handed to the pack / Studio, never
  stored in the workspace) and validate `Origin` (the MCP spec's
  DNS-rebinding guidance); still bound to 127.0.0.1.
- **Audit**: each agent tool call logged (tool, args summary, identity, result
  status) in an in-memory ring + optional `$DTK_HOME/agent/log.jsonl`; no
  credentials, no row values.

### 6. Auth and cost

- `external` / `terminal` packs: the CLI's own auth (subscription login or its
  own key); DTK never sees a credential; cost on the user's existing plan.
- `agent-sdk` / `chat` packs: key read from the environment only
  (`ANTHROPIC_API_KEY`, or an OpenAI-compatible base URL + key for local /
  other models); never written to the workspace, localStorage or logs. Check
  each provider's terms for subscription vs API-key use at implementation.
- Cost visibility: `usage` events (tokens in / out per turn, cumulative per
  session) shown in the panel; an optional per-session cap
  (`DTK_AGENT_MAX_TOKENS`) that stops the loop. Model choice per pack, default
  the provider's current mid-tier model, overridable by env.
- Token economy by design: tools return compact JSON (`Result` minus figures
  unless asked, capped rows), the UI context is small, schemas are fetched on
  demand rather than all at once.

### 7. Dependencies (`mcp` and `claude-agent-sdk` approved 2026-10-02; the others still need approval)

| Candidate | Role | Where | Needed by |
|---|---|---|---|
| `mcp` (official Python MCP SDK) | MCP server, streamable HTTP + stdio | engine, optional extra `agent` | phase 2 (approved 2026-10-02) |
| `claude-agent-sdk` (Python) | `agent-sdk` pack (agent loop, permissions) | engine extra `agent-sdk` | phase 3 (approved 2026-10-02) |
| `anthropic` SDK, or `httpx` promoted from dev to the extra | `chat` pack (Messages / OpenAI-compatible loop) | engine extra | phase 4 |
| `@xterm/xterm` (+ `@xterm/addon-fit`) | terminal panel | `datatoolkit-web` | terminal packs only |
| `websockets` (uvicorn WebSocket support) + `pywinpty` on Windows (stdlib `pty` elsewhere) | PTY bridge | engine extra | terminal packs only |
| none | SSE (`StreamingResponse` + browser `EventSource`) | — | phase 1 |
| user-installed, not deps | `claude`, `gemini`, `opencode` CLIs | user machine | external / terminal packs |

### 8. Phased plan (sub-issues of #9)

0. **Design** — this note approved (doc proposal).
1. **UI bridge** — engine: `/api/ui/context`, `/api/ui/events` (SSE),
   `/api/ui/ack`, token + Origin; Studio: publish the context, execute
   commands through the reducer, ack. Testable with curl + an e2e spec, no
   agent, no new dependency.
2. **MCP server + guard** — `dtk_engine/agent/`, tools over the contract and
   the UI bridge, path guard, row caps, audit log; `dtk-mcp` stdio entry;
   `dtk-mcp config <pack>` for Claude Code / gemini / opencode as
   **external** packs. First user value: Matteo's own CLI drives Studio live.
3. **In-Studio panel + first pack** — panel shell (rail icon, dock or side
   panel), the chat event protocol, the first adapter chosen in its
   sub-issue, `stub` pack + e2e.
4. **More packs** — the other chat adapter (local / OpenAI-compatible) and
   the terminal panel for CLI packs, each opt-in.
5. **Hardening** — usage cap, review-first mode for destructive ops, engine
   revisions if a headless writer is ever needed, Docker image story.

## As built — phase 1 (engine #86, web #81, 2026-10-02)

Engine (`src/dtk_engine/ui_bridge.py`, routes in `http.py`):
- Routes: `PUT/GET /api/ui/context`, `GET /api/ui/events?session=` (SSE),
  `POST /api/ui/ack`, and `POST /api/ui/commands {type, …}` (+ `session` /
  `timeout` query) so curl and the web e2e can post commands without an agent.
- Guard: per-run token (`DTK_UI_TOKEN`, else random, `app.state.ui_bridge.token`),
  allowed `Origin`, `Host` loopback or in `DTK_UI_ALLOWED_HOSTS` (comma
  hostnames; needed behind nginx / Docker, with `DTK_CORS_ORIGINS`).
- Engine-side acks: unknown id → 404; no live listener → `no_studio`; no ack in
  time → `timeout`; listener drops → pending commands `no_studio`.
- SSE streams never end on their own: servers set uvicorn
  `timeout_graceful_shutdown` (`dtk-api`: 2 s).

Studio (`src/state/agentCommands.ts`, `src/bench/agent/AgentBridge.tsx`):
- Acks: `{ok: true, identity}` (frame after the command; `propose_steps` acked
  once the save gate stored the workspace), `{ok: false, error: "stale"}`
  (workspace or `base_identity` differs, re-checked after a review),
  `"rejected"` (review dismissed), `"bad_command: <reason>"`,
  `"save_failed: <msg>"` (applied, PUT failed, still undoable).
- `propose_steps` ops apply in order (an index refers to the list as earlier
  ops left it), `orderSteps` once at the end, one undo entry; a step's
  `target` defaults to `both`, `params` to `{}`.
- Destructive = a `remove` op, or an add / replace of `drop_columns`,
  `filter_rows`, `drop_low_variance`, `drop_correlated`: review banner
  (Apply / Dismiss), nothing applied meanwhile. Others apply at once with a
  12 s "Agent: <summary>" toast + Undo (no-op while a step editor is open).
- Commands run one at a time in arrival order (the second sees the first's
  identity).
- Published `windows[].params` = persisted `toolParams` + `column` (per-column
  tools dist / outliers / target) and `by` (dist split); `open_window` accepts
  the same keys.
- Token handoff: `<meta name="dtk-ui-token">` (Vite plugin in dev when
  `DTK_UI_TOKEN` is set; `dtk-studio` serves it, loopback Host only).

Open for phase 2: a review longer than the 30 s command timeout expires the
command engine-side and blocks the queue meanwhile (`pending_review` ack vs a
longer timeout); bridge off in a plain `npm run dev` + `dtk-api` until
`DTK_UI_TOKEN` is set on both sides.

## As built — phases 2–3 (2026-10-03)

Engine (`src/dtk_engine/agent/`, extra `agent` / `agent-sdk`; details in the
engine README and `docs/agent-chat-protocol.md`):
- `policy.py` (#65, engine #91): path guard under `$DTK_HOME` or workspace-
  referenced paths, `sql` refused, row caps (50 default, 500 max), data framing,
  audit ring + optional `$DTK_HOME/agent/log.jsonl`.
- MCP server (#64, engine #93): `/mcp` on the `dtk-api` app + stdio `dtk-mcp`;
  the per-run URL and token are published in `$DTK_HOME/agent/runtime.json`
  (engine #89).
- Packs (#66, engine #94): `claude-code`, `gemini`, `opencode` (external, only
  the `dtk` server, built-in tools turned off where the CLI allows), `stub`;
  `dtk-mcp config <pack> [--write <dir>]` and `dtk-mcp doctor`.
- Chat (#67, engine #95): hub + adapter per Studio session carried on the SSE /
  POST pair (`/api/ui/agent*`, events `event: agent`); pack `agent-sdk` runs
  `claude-agent-sdk` on the local Claude Code CLI (its own login, or
  `ANTHROPIC_API_KEY`), built-in tools off (`tools=[]`), `setting_sources=[]`,
  only the in-process `dtk` server, `can_use_tool` denies anything else;
  `DTK_AGENT_MAX_TOKENS` cap. Launch: `uv run dtk-api --agent`.
- The 30 s review timeout is solved by an interim `pending_review` ack
  (engine #92): the queue is not blocked by an open review.
- `workspace_rows` takes a view-only filter / sort (engine #90).

Studio (`src/state/agentCommands.ts`, `src/bench/agent/`): the bridge now has a
curated command per user gesture, each undoable (one undo entry) and acked:
`propose_steps`, `open_window`, `select_columns`, `set_view`, `pick_row`,
`pick_cell`, `clear_selection`, `add_variable`, `draft_chart`, `add_chart`,
`edit_step`, `fill_editor`, `set_target`, `set_dist_by`, `set_tool_params`,
`set_grid_view` (web #86–#99); touched columns / windows / steps are
highlighted and each command has its Undo toast (#89); the chat panel
(`src/bench/agent/panel/`, web #100: message list, tool chips, permission /
review prompts, usage line, Stop, "agent unavailable" state).
Not exposed on purpose (Matteo, 2026-10-03, datatoolkit-issues#86): sources,
label join and merges.
Phase 4 (2026-10-03, engine #100–#104, web #107–#111): contract v2 in
`docs/agent-chat-protocol.md`; options route and per-session pack / model
choice (`GET /api/ui/agent/options`); direct API packs `api-anthropic` and
`api-openai` (httpx, no vendor SDK; any OpenAI-compatible or local endpoint);
terminal packs over `WS /api/ui/terminal` (PTY, no shell, same token / Host /
Origin guard, opt-in `--terminal`); read-only attachments under
`$DTK_HOME/uploads`; Markdown replies (`marked` + `dompurify`), collapsed tool
chips, selector and terminal panel, attach button in Studio. The new Studio
commands are MCP tools generated from the published command schema (#100).

## Decisions (Matteo, 2026-10-02 — all recommendations taken)

| Topic | Decision | Sub-issue |
|---|---|---|
| Event channel engine → Studio | SSE (`/api/ui/events`) + `POST /api/ui/ack`, no new dependency | #62 |
| Local endpoint auth | per-run random bearer token + `Origin` check (`/api/ui/*` and `/mcp`), still on 127.0.0.1 | #62 |
| Write model | Studio is the single writer: agent edits relayed, applied by the reducer, undoable; engine-side revisions parked (phase 5) | #63 |
| Confirmation UX | apply at once + Undo toast; destructive ops (remove step, drop rows / columns) reviewed first | #63 |
| MCP library | official `mcp` Python SDK, optional extra `agent`, in the engine repo next to `http.py` | #64 |
| Agent read scope | workspace datasets + any file under `$DTK_HOME`; `sql` sources refused | #65 |
| Export tool | not exposed in phases 2–4 | #65 |
| Default row cap | 50 rows | #65 |
| External packs | Claude Code, gemini and opencode together; generated configs turn the CLI's built-in tools off where possible | #66 |
| First in-Studio pack | chat panel + Claude Agent SDK (`claude-agent-sdk`, extra `agent-sdk`; auth terms checked at implementation); then the provider-agnostic chat backend, terminal pack opt-in last | #67 |

Sub-issues: #62 (phase 1, engine UI bridge), #63 (phase 1, Studio side),
#64 (phase 2, MCP server), #65 (phase 2, policy layer), #66 (phase 2, packs),
#67 (phase 3, panel + first pack), all in matleniz/datatoolkit-issues.
