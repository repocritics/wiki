# simstudioai/sim

A self-hostable workspace for building, deploying, and managing AI agents and workflows visually, conversationally, or in code.

## What it is

Sim is a TypeScript platform where teams assemble AI agents and workflows around a visual builder and a chat surface. It connects agents to large numbers of integrations and every major LLM, and keeps tables, files, knowledge bases, and scheduled tasks in the same workspace so agents have data and memory to work with. It is aimed at teams that want one place to build, run, and monitor agentic automations, either on the hosted service or self-hosted.

## Key features

- Visual workflow builder (ReactFlow) alongside conversational and code-based ways to build agents.
- Stated 1,000+ integrations plus connectors such as Slack, Notion, HubSpot, Salesforce, and databases, and support for every major LLM.
- Built-in data surfaces: structured tables, a shared file store, and synced knowledge bases agents can search.
- Scheduled tasks for recurring agent runs, plus monitoring of runs, logs, schedules, and workflow activity.
- Self-hosting via `npx simstudio` or Docker Compose, including local models through Ollama and vLLM.

## Tech stack

- TypeScript monorepo managed with Turborepo (`turbo` 2.9.14) and the Bun runtime (`bun@1.3.13`).
- Next.js (App Router), pinned via overrides to `next` 16.2.6 with `react`/`react-dom` 19.2.4.
- PostgreSQL with Drizzle ORM (`drizzle-orm` ^0.45.2) and `postgres` ^3.4.5; pgvector required for the database.
- Better Auth for authentication, Zod for schema validation, Shadcn + Tailwind CSS for UI, Zustand and TanStack Query for state.
- Realtime via Socket.io, background jobs via Trigger.dev, remote code execution via E2B, and isolated code execution via isolated-vm.
- Biome for formatting/linting (`@biomejs/biome` 2.0.0-beta.5); Apache-2.0 licensed. Requirements: Bun, Node.js v20+, PostgreSQL 12+ with pgvector.

## When to reach for it

- Building agentic workflows that span many SaaS integrations and LLM providers in one workspace.
- Teams that want a visual builder plus a chat interface rather than writing an orchestration layer from scratch.
- Use cases needing agents backed by structured tables, shared files, and knowledge bases in the same system.
- Self-hosting an agent platform, including with local models via Ollama or vLLM.

## When *not* to reach for it

- You want a lightweight library to drop into an existing app; Sim is a full platform with a Postgres/pgvector dependency and a realtime server.
- Constrained environments — the self-hosting setup path notes a need for substantial memory (around 12GB+ RAM) and several running services.
- The Chat feature is a Sim-managed service; a fully offline, no-external-service deployment of chat requires a Sim-issued Chat API key.

## Maturity signal

Sim has accumulated a large audience since its early-2025 start and shows continuous development, with commits pushed the same week as this writing and a broad, deliberately engineered codebase (extensive internal checks, contract-sync scripts, and agent tooling). The Apache-2.0 license makes it safe to adopt and self-host commercially. The high open-issue count and the many active integration and validation scripts are consistent with a fast-moving, actively maintained project still expanding its surface area rather than one in maintenance mode.

## Alternatives

- Flowise — use instead when you want a lighter, more focused visual LLM/agent flow builder without Sim's broader workspace of tables, files, and scheduled tasks.
- Langflow — use instead when your workflows center on LangChain-style components and you prefer that ecosystem.
- Dify — use instead when you want an LLMOps platform oriented around apps, RAG pipelines, and prompt management.
- n8n — use instead when your primary need is general-purpose workflow automation across apps, with AI as one node type rather than the core.

## Notes

- The repository ships a large set of internal agent skills and editor rules (`.agents/skills`, `.claude`, `.cursor`), including "add-integration", "add-model", and "validate-*" workflows — the project is itself built with heavy agent-assisted tooling.
- Chat is treated as a managed service even for self-hosted instances: you generate a Chat API key at sim.ai and set `COPILOT_API_KEY` locally.
- The `package.json` reports version `0.0.0` because it is the private monorepo root; it is not a meaningful release number.

## Tags

typescript, nextjs, ai-agents, workflow, low-code, no-code, rag, llm, agent-orchestration, self-hosted
