# mindsdb/mindshub

An open-source unified workspace where you delegate whole projects to open or proprietary models and collect finished, shareable results.

## What it is

MindsHub Cowork is a workspace for knowledge workers to delegate multi-step projects — apps, websites, research, analysis, reporting, and scheduled operations — and get back finished, publishable output. You connect your data, route work to any model (open or proprietary), run interchangeable open-source agents, and turn their output into web applications you can publish to a live URL. This repository is the platform superproject: it pulls together the desktop/web app, the agent backend, and the data engine so the whole stack can be built and run from source, and it can run on your machine, in your VPC, or as a hosted app.

## Key features

- A Model Router to switch between frontier models (Claude, GPT, Gemini) and open models (DeepSeek, Qwen, Kimi) without wiring a separate key per provider.
- Interchangeable open-source agent harnesses — Anton (default) and Hermes — swappable from a dropdown.
- A secure data vault that links systems like BigQuery, Postgres, Gmail, Drive, HubSpot, Notion, and Linear, with credentials scoped per connection so agents never see raw keys.
- Artifacts: agent output becomes documents, dashboards, apps, and code that can be published to a live URL.
- Cross-session memory, a reusable skill library, and scheduled tasks.
- Deployment across cloud, VPC, on-prem, air-gapped, and hybrid infrastructure, with a desktop app (macOS `.pkg`, Windows `.exe`), a browser web app, and a build-from-source path.

## Tech stack

- Primary language is reported as Makefile: the repo is a superproject orchestrated through `make`, pinning git submodules (`frontend`, `backend/core_api`, `backend/core_agent`, `backend/data-vault`) to specific commits.
- Desktop app built with Electron (`make dev` / `make watch`) plus a browser web SPA (`make dev-web`); production and packaging via `make build`, `make dist-mac`, `make dist-win`.
- Python backend indicated by the "Python 3.10–3.13" badge; local runtime uses a `cowork-server` uv tool and per-backend `.venv`s, with state under `~/.anton` (provider keys) and `~/.cowork` (database, hermes, projects).
- Containerization present via `docker-compose.yml`, `docker/api.Dockerfile`, `docker/web.Dockerfile`, and an nginx config.
- MCP is listed among the repository topics.
- No recognized root package manifest — the repository is a submodule superproject rather than a single package.

## When to reach for it

- You want a single workspace to delegate end-to-end projects and publish the results as live artifacts, without doing the engineering yourself.
- You need to route the same work across frontier and open models, or swap agent harnesses, without managing many provider keys.
- You want agents to act over connected systems (databases, email, docs, CRM) while keeping credentials scoped and out of the agent's view.
- You need flexible deployment — including VPC, on-prem, or air-gapped — to retain control over infrastructure, models, permissions, and data.
- You want to build and run the full desktop/web/agent/data stack from source.

## When *not* to reach for it

- You only need a code-level agent library; this is a full platform superproject with submodules, an Electron/web app, and a data engine.
- You want a single pinned repository to clone and build in one step, rather than a recurse-submodules superproject with `make`-driven module pinning.
- You need every frontier model and private artifacts for free; those are gated behind the Pro tier per the README's pricing note.
- Your workflow is developer-centric custom pipeline code rather than a no-engineering, delegate-and-collect workspace.

## Maturity signal

The underlying repository dates back to 2018 and was last pushed in July 2026, so there is a long project history behind a much more recently framed product (MindsHub Cowork) from the MindsDB organization; the git links resolve to `mindsdb/minds`, indicating an established org repurposing a mature codebase toward this workspace. The large star and fork counts and very low open-issue count are consistent with an actively maintained, organization-backed project rather than a hobby effort. The MIT license applies to this repository, while bundled submodules carry their own licenses. Read it as backed and actively maintained, though the current product surface is newer than the repository's age implies.

## Alternatives

- Dify — reach for it when you want a self-hosted LLM app builder centered on visual workflows and an agent/plugin ecosystem rather than a delegate-and-collect workspace.
- Open WebUI — use when you mainly want a chat interface over local or open models without the project-delegation, data-vault, and artifact-publishing layers.
- LangChain / LangGraph — use when you are a developer assembling custom agent pipelines in code rather than working in a no-engineering workspace.

## Notes

- Submodules are configured with `ignore = all` and pins move only via `make pin`, so branch work on modules never shows up as superproject changes and the parent `git status` stays clean.
- `make flush` wipes the local runtime and deletes app state in `~/.anton` and `~/.cowork` (including conversations and saved keys), intended for testing the from-scratch install flow or recovering a broken install.

## Tags

ai-agents, llm, model-routing, workspace, electron, mcp, self-hosted, python, open-source
