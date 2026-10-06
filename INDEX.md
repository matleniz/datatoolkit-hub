# INDEX — navigation router (read this first)

> **Protocol (context economy).** Read THIS index, open the ONE file relevant to
> your task, `grep '^#'` on large files. Do NOT load the whole hub. Code lives in
> `~/datatoolkit` (engine) and `~/datatoolkit-web` (the front, Studio); verify facts there (`grep`/`ls`)
> before asserting.

## Topic → file

| You are looking for… | Open |
|---|---|
| Layers, the engine↔front contract, `Result`, sources→ops→keys, import barrier | `ARCHITECTURE.md` |
| What the toolkit will do (families, backlog, status) | `CAPABILITIES.md` |
| Which tools/deps are allowed | `STACK.md` |
| Adding a key | `HOWTO/add-a-key.md` |
| Adding a transform op (workspace step / sklearn) | `HOWTO/add-a-transform.md` |
| Which transform ops exist, what they fit, their params | `TRANSFORMS.md` |
| Adding / switching a front | `HOWTO/add-a-front.md` |
| Sharing / running anywhere: one-line launcher, Docker or uv (no Docker) | `HOWTO/share-and-run.md` |
| In-Studio agent: MCP server over the contract, UI-context protocol, packs, security | `AGENT-BRIDGE.md` |
| Plug your own agent CLI (Claude Code / gemini / opencode) into Studio | `HOWTO/plug-an-agent.md` |
| Try the Studio agent chat (launch, scenarios, checklist) | `HOWTO/try-the-agent.md` |
| Docker internals (GHCR images, compose, nginx, CI smoke) | `HOWTO/run-with-docker.md` |
| Web front "Studio" (screens, HTTP API routes, tests) | `FRONT-WEB.md` |
| What was done when, what's next | `ROADMAP.md` |
| Course section → capability → Linear issue | `COURSE-MAP.md` |
| Past QA runs (historical, findings already filed) | `reports/` |

## Keys (one card each — open only the one you need)

| Key | Category | Card |
|---|---|---|
| `dataset_overview` | analysis | `KEYS/dataset_overview.md` |
| `train_test_check` | analysis | `KEYS/train_test_check.md` |
| `label_join_preview` | analysis | `KEYS/label_join_preview.md` |
| `file_inspect` | analysis | `KEYS/file_inspect.md` |
| `duplicates` | analysis | `KEYS/duplicates.md` |
| `inconsistencies` | analysis | `KEYS/inconsistencies.md` |
| `missing_values` | analysis | `KEYS/missing_values.md` |
| `outliers` | analysis | `KEYS/outliers.md` |
| `preprocessing_advisor` | analysis | `KEYS/preprocessing_advisor.md` |
| `feature_selection` | analysis | `KEYS/feature_selection.md` |
| `column_distribution` | analysis | `KEYS/column_distribution.md` |
| `target_analysis` | analysis | `KEYS/target_analysis.md` |
| `correlations` | analysis | `KEYS/correlations.md` |
| `chart` | analysis | `KEYS/chart.md` |
| `impute_benchmark` | analysis | `KEYS/impute_benchmark.md` |

New key → copy `KEYS/_template.md`, add a row here.
