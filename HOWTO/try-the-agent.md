# How to try the Studio agent chat (manual checklist)

The chat panel runs the agent loop in the engine on your local Claude Code CLI
(`agent-sdk` pack, `AGENT-BRIDGE.md`). Prerequisite: `claude auth status` shows
`loggedIn: true`.

## Launch (two terminals, same token)
```bash
# 1
cd ~/datatoolkit && uv sync --extra agent-sdk
export DTK_UI_TOKEN=$(openssl rand -hex 24); echo $DTK_UI_TOKEN
uv run dtk-api --agent
# 2
cd ~/datatoolkit-web && npm install
export DTK_UI_TOKEN=<same token>
npm run dev                      # http://localhost:5173
```
One process instead: `dtk-studio --agent` (see the engine README).
The panel is the last rail icon; its header shows the pack and the provider.

## Scenarios (real data, real CLI)
- sort by `age` descending and filter `sexM=1`: VIEW ONLY banner, filtered row count.
- plot `age` by a categorical column: chart window, Undo toast.
- open the distribution of `age`; select two columns.
- impute a column with the median: step added at once, Undo works.
- drop a column: Apply / Dismiss banner in Studio; the chat reports the outcome.
- long request, then Stop; the next message works; usage line accumulates.

## Degraded states
- `dtk-api` without `--agent`: "No agent". `CLAUDE_CONFIG_DIR` pointing at an
  empty dir: "claude CLI not logged in". No `DTK_UI_TOKEN` on the web side:
  "Agent bridge off".

## No quota
`DTK_API_BIN=~/datatoolkit/.venv/bin/dtk-api npx playwright test
e2e/issue63-agent-bridge.spec.ts e2e/issue67-agent-panel.spec.ts` (stub pack,
needs the `agent-sdk` extra in that venv).

## Known gaps
No median per group in `impute` (`group_mean` / `group_prev` / `group_interp`
only). Sources, label join and merges are not agent-driven by decision
(datatoolkit-issues#86). Dogfood report 2026-10-03: issues #104–#111.
