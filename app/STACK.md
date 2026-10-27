# Stack — allow-list of known tools

Rule: datatoolkit only uses tools Matteo knows. **Adding a dependency not
listed here needs Matteo's explicit approval first**, then a line here.
This is the web (Studio) list plus the tools shared with the engine; the
engine's Python dependencies are listed in `~/datatoolkit-hub/STACK.md`.

| Tool | Role | Where |
|---|---|---|
| plotly | figures (JSON contract) | engine, fronts |
| React + TypeScript + Vite | web front "Studio" (npm, one lockfile) | repo `datatoolkit-web` (approved 2026-09-27) |
| ESLint | lint of the web front | `datatoolkit-web` gate |
| eslint-plugin-sonarjs | cognitive-complexity ceiling (25) only | `datatoolkit-web` gate (devDependency, approved 2026-10-01, datatoolkit-issues#3) |
| Vitest (4.x) + jsdom | unit tests of the web front; jsdom declared explicitly since vitest 4 no longer pulls it | `datatoolkit-web` gate (approved 2026-09-27; 3 -> 4.1.11 on 2026-10-15, web #151, datatoolkit-issues#213) |
| @vitest/coverage-v8 | unit coverage, `npm run coverage`, no threshold (45 % lines / 41 % branches on 2026-10-15) | `datatoolkit-web` devDependency, same version as vitest (approved 2026-10-15, datatoolkit-issues#214) |
| Playwright | e2e tests with screenshots | `datatoolkit-web` (approved 2026-09-27) |
| @axe-core/playwright | accessibility scan (axe) on the final state of the critical e2e journeys | `datatoolkit-web` devDependency (approved 2026-10-15, Matteo, datatoolkit-issues#241) |
| eslint-plugin-playwright | lint of e2e specs; forbids `waitForTimeout` (no fixed sleeps) | `datatoolkit-web` devDependency (approved 2026-10-15, Matteo, datatoolkit-issues#241) |
| react-grid-layout | Studio dock windows: drag to move + resize handles, snap-to-grid, serialisable layout | `datatoolkit-web` (approved 2026-09-29, MAT-234) |
| uv launcher + GitHub Release asset `studio-latest` | no-Docker share path: `dtk-studio` (datatoolkit-web `launcher/`) run with `uv tool run --from git+…`; Studio build published as `studio-dist.zip` + sha256 on every push to `main` | datatoolkit-web `launcher/`, `studio-release.yml`, `uv-launcher.yml` (approved 2026-10-02, datatoolkit-issues#55; no PyPI, no runtime dep beyond `dtk-engine[api]`) |
| marked + dompurify | Markdown rendering of the agent chat, output sanitised (column names / cell values are untrusted) | `datatoolkit-web` (approved 2026-10-03, Matteo, chat markdown) |
| @xterm/xterm + @xterm/addon-fit | terminal panel: a CLI agent (`claude`, `gemini`, `opencode`) in the browser over a PTY, opt-in pack | `datatoolkit-web` (approved 2026-10-03, Matteo asked for a terminal in the browser) |
| Docker + Docker Compose | engine + Studio images, one-command install (`compose.yml`) | both repos (approved 2026-09-30, MAT-255/256) |
| Base images python:3.12-slim, node:22-alpine, nginx:alpine | engine runtime; web build stage; web static server + `/api` proxy | Dockerfiles (approved 2026-09-30) |
| GitHub Actions + GHCR | PR smoke tests of the images, multi-arch publish of `ghcr.io/matleniz/datatoolkit-{engine,web}` | `.github/workflows/docker.yml` of both repos (approved 2026-09-30) |


Retired: streamlit (first front, repo `datatoolkit-streamlit` archived 2026-10-01).
