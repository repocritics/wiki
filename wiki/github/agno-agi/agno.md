# agno-agi/agno

A Python framework and runtime for building, running, and managing agent platforms you host yourself.

## What it is

Agno is a framework and runtime for agent platforms. You build agents with the Agno SDK, run them as a service through the AgentOS runtime, and manage the platform through the AgentOS web UI. It is aimed at developers who want to own their agent stack — keeping data, memory, and security posture under their own control rather than delegating them to a hosted product.

## Key features

- Production API with 50+ endpoints, plus SSE and websocket support, for building a product on top of your agents.
- Storage for sessions, memory, knowledge, and traces in your own database.
- 100+ prebuilt toolkit integrations (GitHub, Slack, Postgres, and others) and context providers that pull live data from Slack, Drive, wikis, MCP, and custom sources.
- Human-in-the-loop approval that can pause runs for confirmation and block tools requiring admin sign-off.
- Observability through OpenTelemetry tracing, run history, and audit logs.
- JWT-based RBAC with multi-user, multi-tenant isolation, plus interfaces for Slack, Telegram, WhatsApp, Discord, AG-UI, and A2A.

## Tech stack

- Primary language: Python.
- AgentOS runtime typically deployed with Docker, backed by a Postgres database for data and traces, and packaged with an MCP server and a control plane.
- Observability via OpenTelemetry.
- Starter templates for Railway, Docker, AWS, GCP, Azure, Fly, Render, Modal, and Helm.
- License: Apache-2.0. No root dependency manifest was included in the inputs, so specific version constraints are not available here.

## When to reach for it

- Standing up a self-hosted agent platform where you need a REST API, storage, and a management UI rather than a single script.
- Serving multiple users or tenants that require RBAC and isolation out of the box.
- Building products that need approval gates, scheduling, and audit logging around agent runs.
- Deploying the same agent stack across different cloud providers using the provided starter templates.

## When *not* to reach for it

- A quick one-off script or notebook experiment, where a full runtime and control plane are more infrastructure than the task warrants.
- Teams outside the Python ecosystem, since the SDK is Python-first.
- Cases where you specifically want a fully managed, hosted service and do not want to run and maintain the runtime and database yourself.

## Maturity signal

The project has been under development since 2022 and shows a very recent push date, which points to active, ongoing maintenance rather than a dormant or abandoned codebase. A large star count and a substantial open-issue backlog are consistent with a widely used project that is still moving quickly and fielding real-world usage. The Apache-2.0 license is permissive and business-friendly, and the repository is not archived.

## Alternatives

- LangChain / LangGraph — reach for it when you want a broad, established library ecosystem and graph-based orchestration rather than an opinionated hosted runtime.
- CrewAI — reach for it when your focus is multi-agent role/crew coordination and you do not need a full platform layer.
- LlamaIndex — reach for it when the core problem is data ingestion and retrieval over your documents rather than serving and managing an agent platform.

## Notes

- The recommended "get started" path hands a prompt to a coding agent (Claude Code, Cursor, Codex) that clones a starter repo and follows its guide, rather than a manual setup.
- Agno sends a telemetry event per agent run to prioritize model providers; it states prompts, messages, and outputs are never sent, and telemetry can be disabled with `AGNO_TELEMETRY=false`.

## Tags

python, ai-agents, agent-framework, runtime, developer-tools, llm, mcp, self-hosted, rbac, observability, rest-api
