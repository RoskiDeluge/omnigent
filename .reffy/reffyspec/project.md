# Project Context

## Purpose
Omnigent is an open-source **AI agent framework and meta-harness**: a common
orchestration layer over many coding/agent runtimes (Claude Code, Codex, Cursor,
Kimi Code, Kiro, Pi, GitHub Copilot, and user-authored agents). The goals are to
let users:

- Swap or combine harnesses without rewriting agents.
- Supervise and coordinate multiple agents in one shared session (e.g. an
  orchestrator that delegates to coding sub-agents and routes diffs to a
  reviewer from a different vendor).
- Use any model — first-party API key, Claude/ChatGPT subscription, or any
  OpenAI/Anthropic-compatible gateway, or a Databricks workspace.
- Collaborate in real time across devices (terminal, browser, phone, desktop
  app) and run sessions in disposable cloud sandboxes.
- Govern agents with stacking policies (approval gates, spend caps, tool limits).

Status: alpha (`0.3.0.dev0`). Authored under Databricks, Inc.; Apache 2.0.

## Tech Stack

### Backend (Python 3.12+, managed with `uv`)
- **CLI / TUI:** `click`, `rich`, `prompt_toolkit`, `pexpect`, `pyte` (PTY/terminal handling).
- **Server / API:** `fastapi`, `starlette`, `uvicorn`, `httpx`.
- **Agents / LLMs:** `openai` SDK, `mcp` (Model Context Protocol), `tiktoken`.
- **Data:** `sqlalchemy` 2.x + `alembic` migrations, `pydantic` 2.x.
- **Policies:** `cel-expr-python` (CEL for side-effect-free inline policy eval).
- **Ops:** OpenTelemetry exporters/instrumentation, `psutil`, `keyring`, `tomlkit`.
- Ships three version-locked sibling packages: `omnigent`, `omnigent-client`,
  `omnigent-ui-sdk` (released together at one version).

### Frontend (`ap-web/`)
- **React + TypeScript**, built with **Vite**; `tsc -b` for type-checking.
- **TanStack Query**, TipTap editor, Monaco/Shiki, Tailwind (typography), Lobehub UI.
- Also packaged as an **Electron** desktop app (macOS) and an **iOS** wrapper.
- Lint with **oxlint**, format with **prettier**, test with **vitest** + Playwright.

### Tooling
- `uv` for env/deps; `ruff` for lint + format; `pre-commit`; `pyrefly` for types.
- Deploy targets: Docker Compose, Render, Fly.io, Railway, Hugging Face Spaces, Modal.

## Project Conventions

### Code Style
- Python: enforced by `ruff check` and `ruff format` (run `uv run ruff check . && uv run ruff format --check .`); `pre-commit` hooks gate commits.
- Frontend: `oxlint` + `prettier`; strict TypeScript via project references.
- Dependency pins in `pyproject.toml` carry inline comments explaining *why* a
  bound exists — preserve that practice when touching deps.

### Architecture Patterns
- Declarative agents: an agent is a short **YAML file** (prompt, tools, optional
  sub-agents/reviewers). Agents can author other agents. See `docs/AGENT_YAML_SPEC.md`.
- **Meta-harness / executor abstraction:** harnesses (`claude-sdk`,
  `claude-native`, `codex`, `cursor`, `kiro-native`, `openai-agents`, `pi`,
  `antigravity`, `qwen`, `kimi`, `copilot`, …) plug in behind a common runtime.
- Major backend packages under `omnigent/`:
  - `server/` — FastAPI server + web API, multi-user accounts.
  - `runner/`, `runtime/` — agent execution and orchestration.
  - `inner/` — harness executors/bridges (e.g. `*_executor.py`, `*_harness.py`).
  - `tools/`, `client_tools/` — tool definitions and client-side tool tunneling.
  - `llms/` — model/provider integration.
  - `policies/` — stacking governance (server / agent / session levels; builtins for safety & cost).
  - `host/` — local host daemon (registers a machine to run sessions).
  - `sandbox/`, `environments/`, `terminals/` — cloud sandboxes & PTY terminals.
  - `db/`, `stores/`, `entities/` — persistence (SQLAlchemy + Alembic), domain entities.
  - `repl/`, `onboarding/`, `spec/` — CLI REPL, setup/auth flows, agent spec.
- Native harness terminals are OS-sandboxed: `seatbelt` on macOS, `bubblewrap`
  (`bwrap`) on Linux; `tmux` drives native terminals.

### Testing Strategy
- A behaviour change under `omnigent/` ships with a test; a bug fix adds a test
  that fails before the fix. Pure refactors/renames/type-only/dep-bumps are exempt.
- Prefer the smallest covering test. Backend tests mirror source dirs under
  `tests/` (`server/`→`tests/server/`, `runner/`→`tests/runner/`, etc.).
- `tests/integration/` for cross-component behaviour; `tests/e2e/` for full-stack
  live-LLM flows (slow, gateway-bound). **A PR adding user-facing functionality
  must include at least one e2e happy-path test.**
- Frontend: colocated **vitest** `*.test.ts(x)` next to changed code; user-facing
  UI changes also need a **Playwright** test under `tests/e2e_ui/` (enforced by
  the `E2E UI Required` check).
- Run: `uv run pytest` (e2e/live skipped by default); `cd ap-web && npm test`.

### Git Workflow
- Branch from `main`; keep changes focused; include tests/docs when relevant.
- **Conventional Commits** with scopes, e.g. `fix(server): …`, `feat(openapi): …`,
  `docs: …`, `fix(ui): …`.
- **Sign off commits** with `git commit -s` (Developer Certificate of Origin).
- For larger changes, open an issue first to discuss the approach.

## Domain Context
- **Harness:** an underlying agent runtime/CLI (Claude Code, Codex, Cursor, etc.)
  that Omnigent wraps so agents are portable across them.
- **Host:** a machine registered with the server (`omnigent host`) that can run
  sessions; without it the web UI is read/continue-only.
- **Managed hosts:** server-provisioned cloud sandboxes (Modal, Daytona, Islo,
  E2B, CoreWeave, Kubernetes, OpenShell, Boxlite) so no laptop must stay online.
- **Policies:** rules checked on every agent action — allow, block, or pause for
  approval — stacking server-wide → per-agent → per-session (stricter first).
- **Sessions** are collaborative: share (watch + chat live), co-drive (`attach`),
  or fork (`run --fork`). CLI: `omnigent` (alias `omni`); web UI at `:6767`.
- Example agents in `examples/`: **Polly** (multi-agent coding orchestrator),
  **Debby** (dual-head brainstorming, Claude + GPT), **Scribe** (docs orchestrator).

## Important Constraints
- **Python 3.12+** required (also tested on 3.13).
- Native harnesses require `tmux`; Linux additionally requires `bubblewrap`.
- Node.js 22 LTS+ with `npm` needed for `ap-web/` and several harness CLIs.
- The three sibling packages (`omnigent`, `omnigent-client`, `omnigent-ui-sdk`)
  must share one version — release tooling verifies the pins.
- Multi-user auth is gated by `OMNIGENT_AUTH_ENABLED` (on by default in Docker).
- No secrets, internal URLs, customer data, or private config in issues, tests,
  examples, or logs.

## External Dependencies
- **Model providers:** Anthropic, OpenAI, and OpenAI/Anthropic-compatible
  gateways (OpenRouter, LiteLLM, Ollama, vLLM, Azure); Databricks workspaces
  (via the `databricks` extra).
- **Agent CLIs/SDKs:** Claude Code, Codex, Cursor (`cursor-agent`), Kiro CLI,
  Kimi Code, Pi, GitHub Copilot SDK.
- **Cloud sandbox providers:** Modal, Daytona, Islo, E2B, CoreWeave, Kubernetes,
  OpenShell, Boxlite.
- **Protocols/standards:** Model Context Protocol (MCP), OIDC (Google, GitHub,
  Okta, Microsoft) for team SSO, OpenTelemetry for observability.
- **Distribution:** PyPI (`omnigent`), Homebrew tap (`omnigent-ai/tap`), macOS
  desktop app.
