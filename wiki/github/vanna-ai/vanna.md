# vanna-ai/vanna

A Python framework for turning natural-language questions into SQL and streamed data answers, with user-aware permissions and a pre-built chat web component.

## What it is

Vanna lets applications answer questions about a SQL database in natural language, returning SQL, an interactive table, Plotly charts, and a written summary. Version 2.0 is a complete rewrite centered on user-aware agents: identity flows through system prompts, tool execution, and SQL filtering so results respect per-user permissions. It targets teams building data-analytics interfaces, multi-tenant SaaS, and enterprise deployments that need audit trails and access control on top of a text-to-SQL experience. The package ships both a Python backend and a front-end `<vanna-chat>` web component.

## Key features

- Text-to-SQL generation via LLMs with agentic retrieval, returning streamed progress, SQL, tables, charts, and natural-language summaries.
- User-aware layer: a `UserResolver` you implement extracts identity from requests, and tools enforce access via group memberships, with row-level filtering, audit logs, and per-user rate limiting through lifecycle hooks.
- A pre-built `<vanna-chat>` web component that drops into any page and works with React, Vue, or plain HTML using existing cookies/JWTs.
- Server integration for FastAPI and Flask, exposing a streaming `chat_sse` endpoint.
- Extensibility through a `Tool` base class for custom tools, LLM middlewares (caching, prompt engineering), context enrichers, and observability hooks.
- Broad optional integrations for many LLM providers, SQL databases, and vector stores selected via extras.

## Tech stack

- Python, requiring `>=3.9` (project metadata); package version 2.0.2, built with `flit_core`.
- Core dependencies: `pydantic>=2.0.0`, `click>=8.0.0`, `pandas`, `httpx>=0.28.0`, `PyYAML`, `plotly`, `tabulate`, `sqlparse`, `sqlalchemy`, `requests`.
- Optional server extras: Flask (`flask>=2.0.0`, `flask-cors>=4.0.0`) and FastAPI (`fastapi>=0.68.0`, `uvicorn>=0.15.0`).
- Optional database extras include PostgreSQL, MySQL, ClickHouse, BigQuery, Snowflake, DuckDB, Oracle, MSSQL, Hive, and Presto; optional model/vector extras include OpenAI, Anthropic, Gemini, Mistral, Ollama, Bedrock, ChromaDB, Qdrant, Pinecone, Milvus, Weaviate, FAISS, and others.
- A separate TypeScript web component (`frontends/webcomponent`) built with Vite and documented via Storybook.
- Tooling: pytest with asyncio, ruff, mypy.

## When to reach for it

- Building a data-analytics application with a natural-language query interface over an existing SQL database.
- Multi-tenant SaaS that must filter query results per user and enforce group-based permissions.
- Teams that want a ready-made chat UI plus backend rather than building the streaming interface themselves.
- Enterprise environments with audit-log and rate-limiting requirements.
- Integrating a text-to-SQL agent with an existing authentication system (cookies, JWTs, OAuth tokens).

## When *not* to reach for it

- You need an actively maintained upstream — the repository is archived and read-only.
- You do not need the user-aware permission layer or the bundled web UI; the 2.0 design is opinionated around both.
- You are still on the 0.x `VannaBase` training/RAG approach and cannot migrate; 2.0 is a new Agent-based API, though a `LegacyVannaAdapter` exists for wrapping older instances.

## Maturity signal

The project accumulated a large following (over 23,000 stars) over a multi-year life beginning in 2023, and it reached a substantial 2.0 rewrite. However, the repository is archived, meaning it is read-only and no longer accepting issues or contributions despite a relatively recent final push. Interpret it as a mature, feature-complete codebase that has been frozen: usable as-is under its MIT license, but without ongoing maintenance or security updates from the original team.

## Alternatives

- LangChain — use its SQL chains/agents when you want to compose text-to-SQL inside a broader, actively maintained agent toolkit rather than a dedicated product.
- LlamaIndex — reach for it when your natural-language-to-SQL need sits alongside heavier retrieval and indexing over mixed data sources.
- WrenAI — consider it when you want a standalone, actively developed open-source text-to-SQL application with its own UI.

## Notes

The most notable fact is that a project at a fresh, enterprise-focused 2.0 release is archived, so new development has stopped. Vanna 2.0 is a ground-up rewrite that changes the programming model from `VannaBase` methods to an Agent API and from text/dataframe outputs to streamed rich UI components. The `all` extra pulls in an unusually large set of database, LLM, and vector-store dependencies at once, and the chat web component is shipped and versioned separately from the Python package.

## Tags

python, text-to-sql, llm, rag, sql, database, data-visualization, ai-agents, web-component, fastapi
