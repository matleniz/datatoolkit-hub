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
| `GET /workspaces` · `GET/PUT/DELETE /workspaces/{name}` | store; `PUT` body = the workspace without `name` (taken from the path), unknown fields → 422; returns the saved workspace; `DELETE` → 204. Optional `groups: [{id, name, intent?, step_ids, origin?: "auto"\|"manual", produced_by?}]` (step groups, #249): contiguous slices of `steps` in step order, each step in exactly one group; steps without a group get a singleton auto group, empty groups are dropped, and the saved/returned workspace always carries the normalised `groups`. `Group.name` is one line, 1 to 80 characters: a newline or any control / line-separator character (incl. U+2028, U+2029) is refused with a 422 naming the group. Group ids created by a client: `g` + 8 random base36 characters (never `g<digits>`, reserved for engine-made singletons); the engine only checks the regex and uniqueness against present groups, it does not remember deleted ids. 422 `KeyParamsError` messages: `duplicate group id 'X'`, `step 'sN' is in groups 'A' and 'B'`, `group 'G' names unknown step id 'sN'`, `group 'G' is not a contiguous slice in step order: step 'sN' is at position P, expected Q` |
| `GET /workspaces/summaries` | `list_workspace_summaries` (MAT-171) |
| `POST /workspaces/{name}/rename` `{new_name}` | `rename_workspace` (MAT-171) |
| `POST /workspaces/{name}/duplicate` `{new_name}` | `duplicate_workspace` (MAT-171) |
| `POST /workspaces/{name}/export` `{out_dir, overwrite?, formats?}` | `export_workspace`; `formats` = list of `parquet`, `csv`, `ipynb`, `py` (default `["parquet"]`, checked in `workspace/export.py` before any staging dir exists: a non-list or non-string item is a 422 showing the real value, duplicates are dropped keeping first-seen order, #270) → manifest. The manifest lists `groups` (`id`, `name`, `intent`, `steps` = step indices) and each step entry carries `group_id`; `ipynb` renders one markdown + one code cell per step group (a workspace without groups = one per step), `py` the same with `# --- name ---` comment headers; each step's code is preceded by a `# sN · op target` comment (#249). `overwrite` is transactional (#254): the new export is built in a hidden `.export-staging-*` dir inside `out_dir`, the previous owned entries (`processed/`, `states/`, `code/`, manifest) are moved aside to `.export-old-*`, then the new ones are moved in; on any failure the previous export is restored untouched (replay or write error, same error as before); if the restore itself fails the previous export is kept in `.export-old-*` and the error (`ExportRestoreError`) names it. Concurrent exports to the same `out_dir` are not supported |
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
| `PUT /uploads/{filename}` (raw body) → `{path}` | content-addressed under `$DTK_UPLOAD_DIR` (default `$DTK_HOME/uploads`); the write is atomic (temp file in the same dir, then replace; the file mode follows the umask) and an already-stored file is reused only if its bytes equal the body, otherwise it is rewritten (#258). Size cap (#261): default 512 MiB, set with `DTK_MAX_UPLOAD_MB` (a number of MiB > 0; an invalid value makes `create_app()` fail at startup); a body over the cap (checked early on `Content-Length`, and on the real stream) is a 413 `UploadTooLargeError` naming the limit and the variable, and nothing is written. The body is still buffered in memory up to the cap (streaming to the temp file: #298). The base name is cut to 233 UTF-8 bytes (extension kept, deterministic, so the content-addressed path stays stable, #271); a failing write is a 500 `UploadStoreError` with the fixed message `could not store the upload` (no path, no errno; the real error is logged server-side) |
| `PUT/GET /ui/context` · `GET /ui/events?session=&token=` (SSE) · `POST /ui/ack` · `POST /ui/commands` `{type, …}` (`?session=&timeout=`; `timeout` must be a finite number, else 422 `timeout: must be a number, got ...`; finite values clamped to 0..120 as before, #269) | UI bridge, not the contract (`dtk_engine/ui_bridge.py`, datatoolkit-issues#62; `AGENT-BRIDGE.md`) |
| `GET /ui/sessions` · `GET /ui/commands/schema` · `GET /ui/commands/{id}` | live Studio sessions, published Studio command schema (MCP tools are generated from it, #100), command status incl. `pending_review` (#95) |
| `GET /ui/agent?session=` · `POST /ui/agent/send` `{session, text, attachments?}` · `POST /ui/agent/cancel` · `POST /ui/agent/permission` `{session, id, allow}` | agent chat (events `event: agent` on `/ui/events`; wire format in the engine's `docs/agent-chat-protocol.md`) |
| `GET /ui/agent/options[?refresh=1]` · `POST /ui/agent/config` `{session, pack, model?}` | packs and models offered, per-session pack and model (#118) |
| `POST /ui/agent/attachments` `{session, path}` · `GET /ui/agent/attachments?session=` · `DELETE /ui/agent/attachments/{id}?session=` | read-only chat attachments, files under the upload dir only (#121) |
| `WS /ui/terminal?session=&pack=&model=&cols=&rows=&token=` | terminal packs, a CLI in a PTY (opt-in `dtk-api --terminal`; close codes 4401 / 4403 / 4404 / 4409 / 4422, #120) |
| `/mcp` (outside `/api`) | MCP streamable HTTP, same token / Origin / Host guard (#64) |

Request body schemas (OpenAPI, #251) describe what the handlers really accept, with no
validation added: `head_rows`, `offset`, `limit` are read with `int()` (numeric strings
accepted; a non-numeric string, `null`, an array, `nan`/`inf` or `"5.0"`/`"1e3"` as a string is a 422 `KeyParamsError` naming the field, e.g. `head_rows: must be an integer, got 'abc'`, #252);
`overwrite` is any JSON value (truthiness); `name` in the PUT workspace body is ignored (the
path name wins); `role` is `train` | `test`; `version` is an integer >= 0 or null; `workspace`,
`step` and `spec` reference the `Workspace`, `Step` and `SourceSpec` models.

Write ordering (optional, #263): send `X-Dtk-Write-Seq: <integer 0..2^63-1>` on every
`PUT /workspaces/{name}` of a session (suggested `Date.now()*1000 + counter` per tab). The engine
remembers, per workspace name, the highest sequence it accepted (in memory, per `dtk-api`
process, LRU of 512 names; a restart forgets it); a PUT with a **lower** sequence is refused with
**409** `{type: "StaleWriteError", message, detail: {rule: "stale_write", last_seq}}` and nothing
is written; an equal sequence is accepted (idempotent retry), a higher one becomes the new last;
clients drop a 409 `stale_write` save silently. Without the header the PUT behaves as before and
does not touch the remembered sequence; an invalid header (non-numeric, empty, negative, signed,
repeated, more than 19 significant digits) is a 422 `KeyParamsError` naming the header. The guard
covers only this route: documents, notes, column-notes, rename and agent/MCP step writes bypass it;
leading zeros are accepted; delete forgets the sequence, rename forgets both names; edits from
another client with a different sequence scale stay last-write-wins; CORS allows the header.

Errors: `{type, message}`; 404 for `UnknownKeyError`, `UnknownTransformError`,
`WorkspaceNotFoundError`; 422 for `KeyParamsError`, `SourceError`; 413 for `UploadTooLargeError` (`PUT /uploads`); 500 for
`WorkspaceCorruptError` (an unreadable workspace file, #206: invalid JSON or schema, or not valid UTF-8, #257; the listings show such a workspace as a `corrupt` row and keep listing the others; a leading UTF-8 BOM is accepted and removed at the next save, #260). A malformed, non-UTF-8 or empty JSON body gives 422
`KeyParamsError` ("request body is not valid JSON: …", #223). Unknown key,
unknown param and missing column messages list the valid names (capped at 20,
close match first, #231). A params
validation failure gives a concise `message` (`<loc>: <msg>; …`, never the
pydantic dump) plus `details: [{loc, msg, type}]` (MAT-142). A step-group invariant
violation (`PUT /workspaces/{name}` or any route that validates a workspace, #249) adds
`detail: {rule, group_id, step_id}` to that body: `rule` is one of `duplicate_group_id`,
`step_shared`, `unknown_step`, `not_contiguous`, `out_of_order`, `invalid_name`; `group_id` /
`step_id` name the offender and are `null` when the rule has none; other 422s have no
`detail` key and `message` stays human text. Limits: a field error in the same body hides
`detail`; a bad group id format, empty `step_ids`, an over-long name and duplicate step ids
stay plain errors with `loc`; the document/note update routes do not add `detail`. CORS allows the
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
