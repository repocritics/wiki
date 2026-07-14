# Mintplex-Labs/anything-llm

> An all-in-one, self-hostable RAG + AI-agent desktop/server app that turns your documents into a private ChatGPT over any LLM provider.

[GitHub repo](https://github.com/Mintplex-Labs/anything-llm) ·
[Official website](https://anythingllm.com) ·
[License: MIT](https://github.com/Mintplex-Labs/anything-llm/blob/master/LICENSE)

## Overview

AnythingLLM is a full-stack JavaScript application from Mintplex Labs that packages document ingestion, retrieval-augmented generation (RAG), a vector store, multi-user auth, and an agent runtime into a single installable product[^1]. Its pitch is "own your intelligence": run entirely locally with an embedded LLM/embedder/vector DB and zero external calls, or wire it to any of the ~40 supported LLM providers (OpenAI, Anthropic, Ollama, LM Studio, AWS Bedrock, Groq, and so on) and cloud vector databases. Created in mid-2023, it has grown into one of the more widely deployed open-source RAG front-ends, with ~63k stars as of mid-2026 and near-daily commits.

It ships in two distinct distributions that are easy to confuse. The **Desktop app** (Electron, Mac/Windows/Linux) is a single-user, zero-setup binary aimed at individuals who want a private local chatbot over their files. The **Docker/self-hosted server** is the multi-user product: workspace permissioning, the embeddable chat widget, the developer API, and password auth are Docker-only[^2]. Choosing the wrong one is the single most common source of "feature X is missing" confusion.

The defining tension is breadth versus depth. AnythingLLM optimizes for "works out of the box with anything" — every provider, every vector DB, every document type behind one UI. The cost is that the RAG pipeline itself is deliberately generic: fixed-size chunking, cosine similarity retrieval, and a tunable-but-simple similarity threshold. It is an excellent integration and packaging layer, not a research-grade retrieval engine.

## Getting Started

Docker is the recommended path for anything beyond a single desktop user:

```bash
docker pull mintplexlabs/anythingllm

export STORAGE_LOCATION=$HOME/anythingllm && \
mkdir -p $STORAGE_LOCATION && \
touch "$STORAGE_LOCATION/.env" && \
docker run -d -p 3001:3001 \
  --cap-add SYS_ADMIN \
  -v ${STORAGE_LOCATION}:/app/server/storage \
  -v ${STORAGE_LOCATION}/.env:/app/server/.env \
  -e STORAGE_DIR="/app/server/storage" \
  mintplexlabs/anythingllm
```

Then open `http://localhost:3001`, pick an LLM provider + embedder in the onboarding wizard, create a workspace, and drag documents in. The Desktop app requires no commands — download the installer from the website. All configuration (providers, keys, vector DB) is done in-app rather than in config files.

## Architecture / How It Works

The repo is a Yarn monorepo with three runnable services plus two git submodules[^1]:

1. **`server`** — a Node/Express API that owns the application database (Prisma over SQLite by default), auth, workspace/document management, LLM provider calls, and the vector-DB abstraction layer. This is the brain.
2. **`frontend`** — a Vite + React SPA that talks to the server over REST.
3. **`collector`** — a separate Node/Express service that parses and chunks uploaded files (PDF, DOCX, TXT, audio transcription, web scrapes) into text. It is intentionally isolated so document parsing crashes/leaks don't take down the API.
4. **`embed`** and **`browser-extension`** — submodules for the website chat widget and Chrome extension, maintained in their own repos.

The RAG flow: `collector` extracts text → text is split into chunks → chunks are embedded (native ONNX embedder by default, or an external embedder) → vectors land in the configured vector DB (LanceDB embedded by default; Pinecone/Chroma/Qdrant/Weaviate/Milvus/PGVector as alternatives). At query time the server embeds the question, does a top-k similarity search, filters by a similarity threshold, stuffs the surviving chunks into the system prompt, and calls the chat LLM. There is no reranker or query rewriting in the default path.

Agents are invoked with `@agent` in chat and run a tool-use loop (web browsing, web scraping, RAG search, SQL, custom skills). The no-code "Agent Flows" builder and MCP-server compatibility layer let you add tools without writing provider glue. Because every LLM/embedder/vector-DB is behind a common interface, the provider you choose is a runtime setting, not a build-time decision — the flip side is that capabilities differ silently per provider (e.g., tool-calling quality, multimodal support).

## Production Notes

**Desktop vs Docker is a hard fork of features, not a packaging choice.** Multi-user, permissions, the embeddable widget, and the API keys exist only in the Docker/server build. Teams that pilot on Desktop and then need to share it must migrate to Docker; there is no in-place upgrade path between the two.

**Retrieval quality is the top operational complaint, and it is tuning, not a bug.** Default chunking is size-based and context-blind, so answers degrade on tables, code, and long structured documents. The practical levers are: raise/lower the workspace "similarity threshold," increase "max context snippets," pick a stronger embedder than the native default, and pre-clean documents. Expect to iterate — out-of-the-box retrieval on messy corpora is mediocre.

**Vector DB choice has real consequences.** LanceDB (default) is embedded and file-based — great for single-node, but it lives in your storage volume, so back that volume up. Switching vector DBs later requires **re-embedding every document**; there is no live migration between stores.

**Storage volume is the entire state.** For Docker, the mounted `storage` directory holds SQLite, LanceDB, uploaded documents, and cache. Losing or failing to persist that volume loses everything. `SYS_ADMIN` cap is requested for the built-in Chromium-based web scraping in agents.

**Scaling is vertical, not horizontal.** The server assumes a single instance with local SQLite + embedded vector DB. Running multiple replicas behind a load balancer is not a supported topology without moving to external Postgres/vector-DB and accepting that some paths still assume local storage. This is a workgroup tool, not a horizontally-scaled SaaS backend.

**Telemetry is on by default** (anonymous, PostHog); disable with `DISABLE_TELEMETRY=true`[^3]. Even with telemetry off, expect outbound calls to your chosen providers and to `cdn.anythingllm.com` for model mirroring.

## When to Use / When Not

**Use when:**
- You want a private, self-hosted "chat with your docs" for one person or a small team with minimal setup.
- You need to stay provider-agnostic (swap between local Ollama and cloud APIs from a UI).
- You want built-in multi-user, agents, and an embeddable widget without assembling them from libraries.
- Data locality/privacy matters and you want a fully offline option.

**Avoid when:**
- You need best-in-class retrieval (custom chunking, reranking, hybrid/BM25, query rewriting) — build on a framework instead.
- You need a horizontally-scaled, high-concurrency RAG backend for many tenants.
- You want RAG as a library embedded in your own app rather than a standalone application.
- You need fine-grained control over the prompt-construction and retrieval internals.

## Alternatives

- open-webui/open-webui — similar self-hosted all-in-one chat UI, strongest around Ollama; use it when local-model UX and community plugins matter more than document-RAG breadth.
- run-llama/llama_index — a RAG *framework*, not an app; use it when you need to control chunking, retrieval, and reranking in your own codebase.
- langchain-ai/langchain — general LLM orchestration toolkit; use when you're writing the application yourself and want maximal integration surface.
- imartinez/private-gpt — offline-first document Q&A; use when a leaner, Python, fully-local pipeline is preferable to a feature-broad app.
- danny-avila/LibreChat — multi-user self-hosted chat platform; use when the priority is a polished multi-provider chat UI over deep document ingestion.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-06 | Repository created; early RAG chat-with-docs app[^1]. |
| Desktop | 2024 | Cross-platform single-user Electron desktop app introduced. |
| agents | 2024 | `@agent` tool-use runtime and skills added. |
| MCP + Flows | 2025 | MCP-server compatibility and no-code Agent Flows builder. |
| 1.11.x | 2026 | Ongoing releases; model router, memories, scheduled tasks[^4]. |

(Version dates beyond repo creation are approximate — AnythingLLM ships continuously and does not follow strict semver release cadence; see releases for exact tags.)

## References

[^1]: AnythingLLM README and monorepo structure (`frontend`/`server`/`collector`/`docker`/`embed`/`browser-extension`). https://github.com/Mintplex-Labs/anything-llm
[^2]: AnythingLLM documentation — Desktop vs Docker feature differences and self-hosting. https://docs.anythingllm.com
[^3]: Telemetry & Privacy section, README — `DISABLE_TELEMETRY` opt-out, PostHog provider. https://github.com/Mintplex-Labs/anything-llm#telemetry--privacy
[^4]: AnythingLLM feature docs — model router, memories, scheduled tasks, intelligent tool selection. https://docs.anythingllm.com

## Tags

javascript, nodejs, rag, self-hosted, ai-agents, vector-database, llm, local-ai, document-qa, electron, mcp
