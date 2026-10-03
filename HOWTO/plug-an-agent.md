# How to plug an agent CLI into Studio (external packs)

Run your own agent CLI (Claude Code, Gemini CLI, opencode) next to Studio. It
talks to the engine only through the `dtk` MCP server: it sees what Studio
shows (selection, workspace) and proposes steps that appear live in Studio.
Local engine only (`uv`); the Docker image has no `agent` extra.

```bash
uv sync --extra agent
uv run dtk-api                                          # keep it running; open Studio
uv run dtk-mcp config claude-code --write ~/dtk-agent   # prints the command to run
uv run dtk-mcp doctor                                   # engine / token / Studio / CLIs
```

## Packs
| pack | file written | command printed | turned off |
|---|---|---|---|
| `claude-code` | `dtk.mcp.json` | `claude --strict-mcp-config --mcp-config <dir>/dtk.mcp.json --tools '' --allowedTools 'mcp__dtk__*'` | every built-in tool; every other MCP server |
| `gemini` | `.gemini/settings.json` | `cd <dir> && gemini --skip-trust --allowed-mcp-server-names dtk` | shell, file read/write/edit, web fetch/search, subagents, skills, todos (`tools.exclude`); other MCP servers (`mcp.allowed`) |
| `opencode` | `opencode.json` | `cd <dir> && opencode` | every built-in tool and other MCP servers' tools (`permission: {"*": "deny", "dtk_*": "allow"}`) |
| `stub` | none | none | scripted client for tests (no model, no network) |

Flags were checked against claude 2.1.287, gemini-cli 0.50.0, opencode 1.17.18;
each pack module cites the CLI doc it follows.

- gemini: `tools.core: []` is not used, it also denies MCP tools in 0.50.0.
  Project settings only load in a trusted folder, hence `--skip-trust`.
- dtk tools run without a CLI prompt in all three packs; destructive Studio
  edits still wait for the user's review in Studio.

## Wiring
- Default **stdio**: the config launches `<python of the agent env> -m
  dtk_engine.agent.cli` (+ `DTK_HOME` when set). No token in any file; it
  survives `dtk-api` restarts (the stdio server finds the engine through
  `$DTK_HOME/agent/runtime.json` on each call).
- `--http`: the running `/mcp/` URL + `Authorization: Bearer <token>` from the
  runtime file; valid for that engine run only (file written `0600`). Exits 2
  "no running engine (start dtk-api)" when none runs.
- `--write DIR` never overwrites an existing file unless `--force`; no merging
  into existing configs.

## doctor
`dtk-mcp doctor [--json]`: runtime file (path, url, pid), engine reachable,
token valid (`GET /api/ui/context` 200/404 vs 401), Studio connected (a
listening session in `GET /api/ui/sessions`), pack CLIs on `PATH` with
`--version`. Exit 0 when the engine is up with a valid token.

## Limits / your own session
What a CLI cannot turn off (its own settings, memory files such as CLAUDE.md /
GEMINI.md, hooks, model choice, opencode plugins) stays the user's own session.
Data egress: rows and profiles the agent reads go to the CLI's model provider,
capped at 50 rows per call by default (500 max, `dtk_engine.agent.policy`).
