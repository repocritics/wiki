# reworkd/AgentGPT

A browser-based tool for assembling, configuring, and deploying autonomous AI agents that pursue a goal by generating and executing their own tasks.

## What it is

AgentGPT lets you name a custom AI agent, give it a goal, and have it work toward that goal by thinking up tasks, executing them, and learning from the results. It runs as a full-stack web application you can use through the hosted site or self-host locally. The intended user is someone who wants to experiment with autonomous, goal-driven agents without wiring together an agent loop from scratch.

## Key features

- Configure and deploy autonomous agents from the browser, each given a name and a goal.
- Agents decompose a goal into tasks, execute them, and adapt based on results.
- Bundled setup CLI that provisions environment variables, database, backend, and frontend.
- Local self-hosting via Docker Compose covering the MySQL database, FastAPI backend, and Next.js frontend.
- Integrations with OpenAI (required), plus optional Serper (search) and Replicate API keys.
- Internationalized UI with locale files for multiple languages including English, German, Spanish, French, Italian, Croatian, and Hungarian.

## Tech stack

- Primary language: TypeScript.
- Frontend: Next.js 13 with TypeScript, styled with TailwindCSS and HeadlessUI.
- Backend: FastAPI (Python), bootstrapped from FastAPI-template.
- Auth: NextAuth.js.
- ORM: Prisma (frontend/Node side) and SQLModel (backend/Python side).
- Database: MySQL locally; Planetscale referenced for hosted use.
- Schema validation: Zod (TypeScript) and Pydantic (Python).
- LLM tooling: LangChain.
- Runtime requirement: Node.js >= 18.
- Project scaffolding: create-t3-app (T3 stack) plus FastAPI-template.
- Containerization: Docker and Docker Compose; the repo includes a bundled setup CLI (`setup.sh` / `setup.bat`).

## When to reach for it

- You want to try autonomous, goal-seeking AI agents through a ready-made web UI rather than building the loop yourself.
- You want a self-hostable, full-stack reference for an agent application spanning a Next.js frontend and a FastAPI backend.
- You want a T3-stack plus FastAPI codebase to study or fork as a starting point for your own agent product.

## When *not* to reach for it

- You need a maintained, up-to-date project. The repository is archived, so it will not receive further updates or fixes.
- You want a lightweight library to embed agent logic into an existing app. This is a full application with database, frontend, and backend services, not a small dependency.
- You cannot use GPL-3.0. The license is copyleft and may not fit proprietary or differently licensed distributions.
- You need production-grade reliability. Autonomous agents that plan and execute their own tasks are inherently open-ended, and the local setup expects Docker, a MySQL database, and an OpenAI key.

## Maturity signal

AgentGPT reached significant popularity, with over 36,000 stars and more than 9,000 forks, reflecting the wave of interest in autonomous agents in 2023. However, the repository is archived and its last push was in April 2025, so it is effectively frozen: no new features, fixes, or maintenance should be expected. It is best treated as a historical reference or a fork base rather than an actively developed tool. The GPL-3.0 license and the sizable contributor base indicate it was a genuine open-source effort during its active period.

## Alternatives

- Significant-Gravitas/AutoGPT — reach for it when you want the original autonomous-agent project the space grew around and a more CLI/framework-oriented approach.
- yoheinakajima/babyagi — reach for it when you want a minimal, readable task-driven agent loop to learn from rather than a full web app.
- langchain-ai/langchain — reach for it when you want the underlying LLM orchestration library (which AgentGPT itself builds on) to compose your own agent logic.

## Notes

- The project is split across three directories that mirror its architecture: `next` (Next.js frontend), `platform` (FastAPI backend), and `db` (MySQL setup), with a separate `cli` package that automates first-time setup.
- Despite the Prisma-plus-Planetscale references, the Prisma schema ships with both `useMysql.sh` and `useSqlite.sh` helpers, suggesting SQLite is a supported local database option.
- The README's stated tech stack lists both a TypeScript and a Python validation/ORM layer (Zod/Prisma and Pydantic/SQLModel), reflecting the dual-language full-stack design.

## Tags

typescript, python, llm, ai-agents, autonomous-agents, nextjs, fastapi, langchain, web-application, t3-stack, openai, self-hosted
