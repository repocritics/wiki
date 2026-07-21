# onyx-dot-app/onyx

An open-source, self-hostable AI platform that combines chat, retrieval-augmented generation over your own data, and agents that work with any LLM.

## What it is

Onyx is an application layer over LLMs that provides a feature-rich chat interface plus enterprise search across connected data sources. It targets individuals and teams who want a self-hosted assistant with retrieval-augmented generation, custom agents, and actions grounded in their own documents. It works with both self-hosted models (Ollama, LiteLLM, vLLM) and proprietary providers (Anthropic, OpenAI, Gemini).

## Key features

- Agentic RAG built on a hybrid vector-plus-keyword index for search and answer generation.
- A multi-step deep-research flow that produces in-depth reports.
- Custom agents with their own instructions, knowledge, and actions, plus actions and MCP for interacting with external applications.
- Over 50 indexing connectors out of the box, or connection via MCP.
- Web search (Serper, Google PSE, Brave, SearXNG, and others), an in-house crawler with Firecrawl/Exa support, sandboxed code execution, artifact generation, image generation, and voice mode.
- Two deployment options — Lite (a lightweight chat UI under 1GB memory) and Standard (full RAG stack with background workers, model inference servers, Redis, and MinIO).

## Tech stack

- Python backend requiring Python 3.13, built on FastAPI 0.133.1, Pydantic 2.11.7, SQLAlchemy 2.0.50, and Alembic migrations.
- Celery 5.5.1 with Redis for background job queues and workers.
- LLM access via litellm 1.93.0 and the openai, google-genai, cohere, and voyageai clients.
- OpenSearch (`opensearch-py` 3.2.0) for indexing, with sentence-transformers, torch, and transformers running in a separate model server.
- A TypeScript/Next.js frontend and desktop/widget workspaces managed with Bun 1.3.13.
- Deployable via Docker, Kubernetes, and Helm/Terraform; enterprise features include SAML/OIDC SSO, SCIM, and RBAC.

## When to reach for it

- Standing up a self-hosted AI chat and search assistant over internal company knowledge from many sources.
- Teams that need enterprise controls such as SSO, RBAC, analytics, and query-history auditing on their AI usage.
- Use cases that need RAG plus agents, web search, and code execution in one platform rather than assembled separately.
- Staying model-agnostic across self-hosted and proprietary LLM providers.

## When *not* to reach for it

- A minimal single-user chat client, where the Standard stack's workers, index, and inference servers are more than needed (Lite mode partly addresses this).
- Environments that cannot run the required Python 3.13, OpenSearch, Redis, and model-server components.
- Teams wanting a fully hosted service with no operational burden, unless they use Onyx Cloud.
- Purely offline single-document Q&A that does not warrant connectors and indexing infrastructure.

## Maturity signal

Onyx has over 31,000 stars, dates back to 2023, and is pushed to within the last day, indicating an actively developed and adopted platform. The dual Community/Enterprise editions, extensive connector list, and large Alembic migration history point to a substantial, production-oriented codebase rather than a prototype. The core is MIT-licensed, with enterprise features layered on top.

## Alternatives

- LibreChat — use instead when you mainly want a multi-provider chat UI without built-in enterprise search and connectors.
- Open WebUI — use instead for a lighter self-hosted chat front end, especially over local models, without a full RAG connector stack.
- Dify — use instead when you want a low-code builder for LLM apps and workflows rather than a ready-made chat-and-search product.
- RAGFlow — use instead when deep document-parsing RAG question answering is your only requirement.

## Notes

The repository ships two editions under one license field: an MIT-licensed Community Edition covering chat, RAG, agents, and actions, plus an Enterprise Edition with extra features, which is why GitHub reports the license as unrecognized. The `pyproject.toml` is unusually heavily commented, pinning specific dependency versions to work around named upstream bugs — for example capping `transformers` to avoid all-NaN embeddings caused by a `from_pretrained()` buffer-corruption issue.

## Tags

python, rag, llm, enterprise-search, self-hosted, information-retrieval, nextjs, vector-search, ai-chat, fastapi, agents
