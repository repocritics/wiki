# sickn33/agentic-awesome-skills

A local, agent-owned control plane and 1,900+ skill catalog for composing, validating, and planning skill stacks for coding agents.

## What it is

AAS Core (Agentic Awesome Skills) is a locally run system that lets a coding agent search a large catalog of agent skills, choose exact skill IDs, and record that choice as a reproducible, reviewable stack. Rather than ranking or recommending, its read-only MCP server exposes catalog search and inspection, and a CLI validates the agent's selection and produces an immutable per-target plan before any changes are applied. It targets developers using Codex, Claude, Cursor, Gemini CLI, and similar agents who want inspectable, version-pinned skill selections instead of opaque one-shot installs.

## Key features

- A read-only local MCP server with `search_skills`, `get_skill`, `compose_stack`, `inspect_stack`, `diff_stack`, and selection-evidence tools; it does not rank, recommend, or hide skills.
- An `aas` CLI that persists `aas-stack.json`, validates a proposed stack, and produces an immutable plan preview without applying changes.
- A catalog of 1,969+ skills spanning development, testing, security, infrastructure, product, and marketing.
- A browser-local Workbench that reviews stack and plan JSON in memory without filesystem access or installation.
- Multiple distribution paths: full-library install (`npx agentic-awesome-skills`), specialized domain plugins, bundles, and workflows.
- Host-specific install flags for Claude Code, Cursor, Gemini CLI, Codex, Antigravity, Kiro, OpenCode, Copilot, and more.

## Tech stack

- Primary language Python for the tooling scripts (validation, indexing, catalog build, auditing).
- Node.js >=22 for the CLIs (`agentic-awesome-skills`, `aas`, `aas-mcp`); npm package v15.1.0.
- Runtime dependencies `ajv` (^8.20.0) for schema validation, `sanitize-filename`, and `yaml` (^2.9.0), all bundled.
- Versioned stack and result JSON schemas under `schemas/aas-v1`; generated catalog and index data under `data/`.
- A companion web app (`apps/web-app`) built with Vite, React, and TypeScript and hosted on GitHub Pages.

## When to reach for it

- You want an agent's skill selection turned into durable, version-pinned, reviewable state (`aas-stack.json` plus an immutable plan).
- You need a large searchable catalog and want the agent, not a ranking service, to choose skills.
- You want to review or validate a proposed stack before anything is installed.
- You install skills across several different agent hosts and want one source with per-host targets.

## When *not* to reach for it

- You want an agent to actually apply and manage installed skills transactionally — apply and recovery are explicitly experimental and outside the supported path.
- You need certified semantic fit or compatibility; Core validates only structural and identity validity, not suitability.
- You only need a handful of Obsidian- or domain-specific skills; a smaller curated set may be simpler.
- Your host cannot run a local stdio MCP server or Node >=22.

## Maturity signal

Created in mid-January 2026 and pushed within days of this writing, the project is actively developed with heavy tooling, CI, and frequent releases (currently the 15.x line). The maintainers are candid that AAS Core is an "agent-first preview": catalog search, composition, validation, and planning are supported, while apply and recovery remain experimental. MIT licensing, a large star count, and extensive validation and audit scripts indicate a serious, fast-moving effort that is still explicit about its trust boundaries.

## Alternatives

- kepano/obsidian-skills — use it when you specifically want Obsidian-focused skills rather than a broad multi-domain catalog.
- Awesome Claude Skills — use a curated awesome-list when you want a hand-picked shortlist rather than a large searchable control plane (the README contrasts the two directly).
- Direct `npx` or plugin skill installs — use these when you already know the exact skill IDs and do not need stack recording or planning.

## Notes

The design deliberately separates "the agent chooses" from "the system records and validates": Core never ranks or recommends, and manifests have a technical maximum of 128 skills. The README is emphatic that structural and identity validity does not certify semantic fit, setup correctness, or safety to apply — an unusually explicit scoping of what the tool does and does not guarantee.

## Tags

python, nodejs, agent-skills, mcp, cli, developer-tools, catalog, ai-coding, skill-library, llm
