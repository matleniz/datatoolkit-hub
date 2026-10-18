# HTTP API — `dtk-api` (engine, `dtk_engine/http.py`, extra `api`)

Moved here from `FRONT-WEB.md` on 2026-10-15 when the web docs left this hub
(they live in `~/datatoolkit-web-hub`). Any front talks to the engine only
through these routes.

Base `/api`. Bodies and responses are the contract's JSON, unchanged. Every
JSON request body is described in `/openapi.json` (`WorkspaceBody` is generated
from the `Workspace` model; datatoolkit-issues#229, engine #162). The handlers
read the raw body, so the schema documents the shape without changing what is
accepted (integers given as strings are still coerced).

| Route | Contract |
|---|---|
| `GET /keys` · `GET /keys/{id}/schema` · `POST /keys/{id}/run` `{params}` | `list_keys`, `key_schema`, `run_key` |
| `GET /transforms` · `GET /transforms/{op}/schema` | `list_transforms`, `transform_schema` |
| `GET /workspaces` · `GET/PUT/DELETE /workspaces/{name}` | store; `PUT` body = the workspace without `name` (taken from the path), unknown fields → 422; returns the saved workspace; `DELETE` → 204 |
| `GET /workspaces/summaries` | `list_workspace_summaries` (MAT-171) |
| `POST /workspaces/{name}/rename` `{new_name}` | `rename_workspace` (MAT-171) |
| `POST /workspaces/{name}/duplicate` `{new_name}` | `duplicate_workspace` (MAT-171) |
| `POST /workspaces/{name}/export` `{out_dir, overwrite?, formats?}` | `export_workspace`; `formats` = list of `parquet`, `csv`, `ipynb`, `py` (default `["parquet"]`, checked in `workspace/export.py`) → manifest |
| `GET /workspaces/{name}/documents` · `POST /workspaces/{name}/documents` `{path, name?, note?}` · `DELETE /workspaces/{name}/documents/{doc_id}` | `list_documents`, `add_document`, `remove_document` |
| `GET /workspaces/{name}/documents/{doc_id}/text?offset=&max_chars=` · `GET /workspaces/{name}/documents/{doc_id}/file` (file response) | `document_text`; the stored document file |
| `POST /documents/describe` `{path, name?}` | `describe_document` (before attaching) |
| `POST /source/columns` `{spec}` | `source_columns` |
| `POST /workspace/preview` `{workspace, role, head_rows}` | `preview_workspace` |
| `POST /workspace/rows` `{workspace, role, version, offset, limit, columns?, filter?, sort?}` | `workspace_rows` (MAT-127; optional `columns` filter MAT-152; view-only `filter` / `sort`, `total_unfiltered`, datatoolkit-issues#87) |
| `POST /workspace/profiles` `{workspace, role, version, columns?}` | `column_profiles` (optional `columns` filter MAT-152) |
| `POST /workspace/preview-step` `{workspace, step, role}` | `preview_step` |
| `POST /workspace/align` `{workspace}` | `align_report` |
| `POST /workspace/column-notes` `{workspace, role="train", version?}` → `{notes, keys}` | column notes of a role (422 when `workspace` is missing) |
| `PUT /uploads/{filename}` (raw body) → `{path}` | content-addressed under `$DTK_UPLOAD_DIR` (default `$DTK_HOME/uploads`) |
| `PUT/GET /ui/context` · `GET /ui/events?session=&token=` (SSE) · `POST /ui/ack` · `POST /ui/commands` `{type, …}` (`?session=&timeout=`) | UI bridge, not the contract (`dtk_engine/ui_bridge.py`, datatoolkit-issues#62; `AGENT-BRIDGE.md`) |
| `GET /ui/sessions` · `GET /ui/commands/schema` · `GET /ui/commands/{id}` | live Studio sessions, published Studio command schema (MCP tools are generated from it, #100), command status incl. `pending_review` (#95) |
| `GET /ui/agent?session=` · `POST /ui/agent/send` `{session, text, attachments?}` · `POST /ui/agent/cancel` · `POST /ui/agent/permission` `{session, id, allow}` | agent chat (events `event: agent` on `/ui/events`; wire format in the engine's `docs/agent-chat-protocol.md`) |
| `GET /ui/agent/options[?refresh=1]` · `POST /ui/agent/config` `{session, pack, model?}` | packs and models offered, per-session pack and model (#118) |
| `POST /ui/agent/attachments` `{session, path}` · `GET /ui/agent/attachments?session=` · `DELETE /ui/agent/attachments/{id}?session=` | read-only chat attachments, files under the upload dir only (#121) |
| `WS /ui/terminal?session=&pack=&model=&cols=&rows=&token=` | terminal packs, a CLI in a PTY (opt-in `dtk-api --terminal`; close codes 4401 / 4403 / 4404 / 4409 / 4422, #120) |
| `/mcp` (outside `/api`) | MCP streamable HTTP, same token / Origin / Host guard (#64) |

Errors: `{type, message}`; 404 for `UnknownKeyError`, `UnknownTransformError`,
`WorkspaceNotFoundError`; 422 for `KeyParamsError`, `SourceError`; 500 for
`WorkspaceCorruptError` (an unreadable workspace file, #206). A malformed, non-UTF-8 or empty JSON body gives 422
`KeyParamsError` ("request body is not valid JSON: …", #223). Unknown key,
unknown param and missing column messages list the valid names (capped at 20,
close match first, #231). A params
validation failure gives a concise `message` (`<loc>: <msg>; …`, never the
pydantic dump) plus `details: [{loc, msg, type}]` (MAT-142). CORS allows the
Vite dev origin (localhost / 127.0.0.1 :5173) plus `DTK_CORS_ORIGINS`
(comma-separated). `dtk-api --port 8765` runs uvicorn; `--port` outside 1..65535 is a usage error,
exit 2 (#225). Without the `api` extra, `dtk-api` prints the install command
and exits 1 (#224).

**UI bridge auth (datatoolkit-issues#62, #63).** The `/ui/*` routes need the
per-run token (`Authorization: Bearer`, or `?token=` for `EventSource` and the
terminal WebSocket, as browsers cannot set headers on either; engine side
`DTK_UI_TOKEN` or a random token at `app.state.ui_bridge.token`), an allowed
`Origin` when present and a loopback or `DTK_UI_ALLOWED_HOSTS` `Host`; the
contract routes stay token-free. Semantics: `AGENT-BRIDGE.md` → "As built".

## Run it
`cd ~/datatoolkit && uv run --extra api dtk-api --port 8765`.
