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
- No RAG or auto-pilot; the user drives, the agent assists. A small curated
  per-workspace memory exists since #179 (below), not a retrieval system.
- Docker: the in-Studio chat is opt-in in the image since 2026-10-04 (`DTK_AGENT=1`, token required, API packs only); no CLI packs, terminal or `/mcp` there (`HOWTO/run-with-docker.md`).
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
| `preview_steps`, `evaluate` | same | dry run of a LIST of steps; stats of formula expressions at a version or after draft steps (#155) — the agent's scratch space, nothing reaches Studio |
| `get_notes` | `column_notes` + workspace | workspace / step (by id) / column notes, renames followed (#152) |
| `set_note` | UI bridge → Studio | write or clear a note, one undo entry (#152) |
| `export_workspace` | same | formats ipynb (default) / py / csv / parquet, only into `$DTK_HOME/exports/<workspace>`, no path from the agent (#156) |
| `get_ui_context` | UI bridge | also published as MCP resource `studio://context` |
| `propose_steps` | UI bridge → Studio | add / replace / remove steps; Studio applies through the reducer (undoable) and acks |
| `open_window`, `select_columns`, `set_view` | UI bridge → Studio | `ToolId` + params, columns, role / version |

  Not exposed: `save_workspace`, `delete_workspace`, `rename_workspace`,
  `duplicate_workspace`, uploads. `export_workspace` is exposed since #156, only
  into `$DTK_HOME/exports/<workspace>`.
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
| `id` | `claude-code`, `agent-sdk`, `api-anthropic`, `api-openai`, `opencode`, `gemini`, `stub` |
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
- **chat** (`agent-sdk`, `api-anthropic`, `api-openai`): Studio panel
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
  a local model through the `api-openai` pack keeps data on the machine.
- **Local endpoint hardening**: `/mcp` and `/api/ui/*` require a per-run
  random bearer token (generated at start, handed to the pack / Studio, never
  stored in the workspace) and validate `Origin` (the MCP spec's
  DNS-rebinding guidance); still bound to 127.0.0.1. Browsers cannot set
  headers on `EventSource` / WebSocket: `/api/ui/events` and `/api/ui/terminal`
  take the token as `?token=`, so uvicorn's logs redact it (`RedactTokenFilter`).
  Attachment bytes go through the contract route `PUT /api/uploads` (no token,
  content-addressed); only `POST /api/ui/agent/attachments` (registration) is
  guarded. The agent path guard refuses `$DTK_HOME/agent` (UI token, audit
  log, terminal configs) even through symlinks or workspace references.
- **Audit**: each agent tool call logged (tool, args summary, identity, result
  status) in an in-memory ring + optional `$DTK_HOME/agent/log.jsonl`; no
  credentials, no row values.

### 6. Auth and cost

- `external` / `terminal` packs: the CLI's own auth (subscription login or its
  own key); DTK never sees a credential; cost on the user's existing plan.
- `agent-sdk` / `api-anthropic` / `api-openai` packs: key read from the environment only
  (`ANTHROPIC_API_KEY`, or an OpenAI-compatible base URL + key for local /
  other models); never written to the workspace, localStorage or logs. Check
  each provider's terms for subscription vs API-key use at implementation.
- Cost visibility: `usage` events (tokens in / out per turn, cumulative per
  session) shown in the panel; an optional per-session cap
  (`DTK_AGENT_MAX_TOKENS`) that stops the loop. Model choice per pack, default
  the provider's current mid-tier model, overridable by env: `api-anthropic`
  takes the first `/v1/models` id containing `sonnet` (`DTK_ANTHROPIC_MODEL`),
  `api-openai` the first listed (`DTK_OPENAI_MODEL`).
- Token economy by design: tools return compact JSON (`Result` minus figures
  unless asked, capped rows), the UI context is small, schemas are fetched on
  demand rather than all at once.

### 7. Dependencies (all approved: `mcp`, `claude-agent-sdk` 2026-10-02; `httpx`, `websockets`, `marked` + `dompurify`, `@xterm/xterm` 2026-10-03, see `STACK.md`)

| Candidate | Role | Where | Needed by |
|---|---|---|---|
| `mcp` (official Python MCP SDK) | MCP server, streamable HTTP + stdio | engine, optional extra `agent` | phase 2 (approved 2026-10-02) |
| `claude-agent-sdk` (Python) | `agent-sdk` pack (agent loop, permissions) | engine extra `agent-sdk` | phase 3 (approved 2026-10-02) |
| `httpx` (promoted from dev to the extras), no vendor SDK | `api-anthropic` / `api-openai` packs (Messages / OpenAI-compatible loop) | engine extras | phase 4 |
| `@xterm/xterm` (+ `@xterm/addon-fit`) | terminal panel | `datatoolkit-web` | terminal packs only |
| `websockets` (uvicorn WebSocket support); stdlib `pty`, POSIX only (no `pywinpty`) | PTY bridge | engine extras | terminal packs only |
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
   revisions if a headless writer is ever needed, Docker image story
   (done 2026-10-04: engine #112, web #123; revisions parked on purpose).

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
Studio modules: `src/bench/agent/panel/` (chat, picker, Markdown),
`src/bench/agent/terminal/` (xterm, socket, 44xx close codes),
`src/bench/agent/attachments/` (upload, then register).

Audit hardening (2026-10-04, engine #105–#111, web #113–#118): the path guard
refuses `$DTK_HOME/agent` (#132, supersedes "any file under `$DTK_HOME`" below);
the `?token=` is redacted in uvicorn logs (#134, #135); terminal I/O
back-pressure with a bounded output queue (#133); a chat session with no SSE
listener for 5 min (`IDLE_GRACE`) is reaped: turn cancelled, adapter closed
(the `claude` CLI exits), conversation, usage and attachments dropped (#136);
API packs bound the resent history (latest 4 turns whole, older tool results
elided, oldest turns dropped over a size cap, one retry on "context too long",
#137); `read_attachment` reads at most the text limit + 1 byte (#138); one
`dtk_home()` resolver (#139). Studio: an attachment removed while uploading is
detached (#128), session attachments shown and detachable (#129), the dev
server reads the token from `runtime.json` when it points at the proxied engine
(#98), streaming Markdown re-rendered at most every 100 ms (#131), one agent
HTTP plumbing (#130), a contract test of the command parser against
`/api/ui/commands/schema` (#103).

Docker (2026-10-04, datatoolkit-issues#147, #148; engine #112, web #123): the
engine image installs the extra `agent`; `DTK_AGENT=1` starts `dtk-api --agent
<pack>` with `api-anthropic` / `api-openai` / `stub` only and refuses to start
without `DTK_UI_TOKEN` (exit 64). The web container injects the token meta at
start (`docker/40-dtk-ui-token.sh`); nginx streams the SSE, masks `token=` in
its log and returns 404 for `/mcp` and the terminal. Loopback port only.

## As built — workspace documents and agent memory (engine #133, #134; web #130, #131, 2026-10-06)

- **Documents** (#178, Matteo: Sources card section, PDF included):
  `Workspace.documents` (≤ 50) = `{id: d<n>, name, path (upload ref,
  content-addressed, never copied), mime, size, kind: text|pdf|table|other,
  added_at, note?}`. Every read re-resolves the path under the upload dir
  (symlinks followed; outside → refused). Text ≤ 1 MB; PDF text through the
  optional extra `pdf` (`pypdf`, ≤ 50 MB / 500 pages; without it a PDF is
  listed, not read); a table document is read with the usual tools on its
  `source` spec. Agent: `list_documents`, `read_document` (sliced, framed as
  data), UI command `keep_attachment` (a chat attachment becomes a document,
  ack carries `document_id`). Export lists documents in the manifest.
- **Agent memory** (#179, Matteo: in the workspace JSON, agent writes freely
  with toast + Undo): `Workspace.memory` = `{id: m<n>, text ≤ 500, kind:
  fact|decision|preference|todo, updated_at}`, ≤ 100 entries / 8,000
  characters. UI commands `remember {text, kind?, memory_id?}` / `forget
  {memory_id}` (field `memory_id`, `id` is the relay's), tool `get_memory`.
  Documents and memory are injected once per conversation and workspace,
  after the `[Studio: …]` note. Studio: Memory view in the agent panel (list,
  edit, delete, clear); every change is one Undo entry.

## As built — epic #160, parkison dogfood fixes (engine #118, #119, #125, #127, 2026-10-06)

- **Preview size** (#154): `preview_step` returns an object (never JSON in a
  string); `changed`, `removed_rids`, fitted `state` cut to 20 items (200 with
  `detail: true`), column lists to 200, `elided` gives full lengths. An
  oversize result is shrunk list by list; `DTK_AGENT_MAX_CHARS` default 50k.
- **Stable step ids** (#153): `Step.id` = `s` + opaque text, kept on replace;
  a workspace without ids reads `s<position>` deterministically until saved.
  Ids never enter the data identity nor the cache keys. `propose_steps` ops
  `{replace: {id, step}}`, `{remove: {id}}`, `{add}` (appends); the tool layer
  fills `base_steps {id: {op, target, params}}` the agent last saw. Rule
  (Matteo): **rebase by id** — applies while every targeted id exists and is
  unchanged, else ack `stale: [{id, reason: removed|changed}]`; acks carry
  `added_ids`. Index ops keep the old base_identity rule (legacy).
  Every tool result carries `identity` and, after a user-side edit,
  `workspace_changes` (added / removed / changed ids, reviewed proposals).
- **Context economy** (#151): each turn is prefixed with a `[Studio: …]` note
  (view, identity, full step list on the first turn, then only the changes);
  `get_profiles` (no `columns`), `align_report`, `list_keys`,
  `list_transforms` compact unless `detail: true`. `usage` events split
  `uncached_input_tokens` / `cache_creation_input_tokens` /
  `cache_read_input_tokens` / `output_tokens` + `context_tokens`. History
  compaction for the agent-sdk pack (Matteo): CLI auto-compact via
  `CLAUDE_CODE_AUTO_COMPACT_WINDOW`, default ~60k, `DTK_AGENT_COMPACT_AT`.
- **Scratch space** (#155, Matteo: stateless, no draft branch):
  `preview_steps` (≤ 50 steps, in memory, per-step state) and `evaluate`
  (≤ 20 expressions, `where` row filter, `count, missing, mean, std, min,
  q25, median, q75, max, sum`). The system prompt forbids temporary steps.

## Decisions (Matteo, 2026-10-02 — all recommendations taken)

| Topic | Decision | Sub-issue |
|---|---|---|
| Event channel engine → Studio | SSE (`/api/ui/events`) + `POST /api/ui/ack`, no new dependency | #62 |
| Local endpoint auth | per-run random bearer token + `Origin` check (`/api/ui/*` and `/mcp`), still on 127.0.0.1 | #62 |
| Write model | Studio is the single writer: agent edits relayed, applied by the reducer, undoable; engine-side revisions parked (phase 5) | #63 |
| Confirmation UX | apply at once + Undo toast; destructive ops (remove step, drop rows / columns) reviewed first | #63 |
| MCP library | official `mcp` Python SDK, optional extra `agent`, in the engine repo next to `http.py` | #64 |
| Agent read scope | workspace datasets + any file under `$DTK_HOME` except `$DTK_HOME/agent` (since #132); `sql` sources refused | #65 |
| Export tool | not exposed in phases 2–4; exposed since 2026-10-06 into `$DTK_HOME/exports/<workspace>` only (#156) | #65 |
| Default row cap | 50 rows | #65 |
| External packs | Claude Code, gemini and opencode together; generated configs turn the CLI's built-in tools off where possible | #66 |
| First in-Studio pack | chat panel + Claude Agent SDK (`claude-agent-sdk`, extra `agent-sdk`; auth terms checked at implementation); then the provider-agnostic chat backend, terminal pack opt-in last | #67 |

Sub-issues: #62 (phase 1, engine UI bridge), #63 (phase 1, Studio side),
#64 (phase 2, MCP server), #65 (phase 2, policy layer), #66 (phase 2, packs),
#67 (phase 3, panel + first pack), all in matleniz/datatoolkit-issues.
