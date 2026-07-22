# Tencent/WeKnora

A self-hostable, Go-based LLM knowledge platform that turns documents into RAG-based Q&A, an autonomous ReAct agent, and a self-maintaining wiki.

## What it is

WeKnora is an open-source knowledge framework for document understanding, semantic retrieval, and autonomous reasoning. It ingests documents from multiple sources and exposes them through three modes: RAG-based quick Q&A, a ReAct agent that orchestrates retrieval, MCP tools, and web search for multi-step tasks, and a Wiki mode that distills raw documents into interlinked markdown with a knowledge graph. It targets teams that need a self-hosted, multi-tenant knowledge assistant with control over their data.

## Key features

- Three interaction modes: RAG Q&A, a ReAct agent for multi-step reasoning, and agent-generated Wiki pages with a knowledge graph.
- Multi-source ingestion from Feishu, Notion, Yuque, and RSS, and support for 10+ document formats including PDF, Word, Excel, images, EPUB, and MHTML.
- Swappable backends: 20+ LLM providers, multiple embedding providers, and vector stores including pgvector, Elasticsearch, OpenSearch, Milvus, Weaviate, Qdrant, Apache Doris, and Tencent VectorDB.
- Multiple delivery surfaces: Web UI, REST API, the `weknora` CLI, a Chrome extension, an embed widget, and a WeChat Mini Program, plus IM channels such as WeCom, Feishu, Slack, Telegram, DingTalk, and Mattermost.
- Enterprise controls: workspace RBAC with a four-tier role matrix, per-resource ownership, per-workspace audit log, scoped API keys, and AES-256-GCM encryption for credentials.
- Retrieval strategies including BM25 sparse, dense, GraphRAG, parent-child chunking, and HNSW-accelerated pgvector, with Langfuse observability and a runtime task-queue dashboard.

## Tech stack

- Go 1.26.0 (module `github.com/Tencent/WeKnora`).
- Gin for the web layer and GORM over PostgreSQL/SQLite; pgvector for vector storage.
- asynq with Redis (`go-redis`) for the async task queue; OpenTelemetry for tracing.
- Provider and vendor SDKs including go-openai, the Lark/Feishu SDK, Milvus, Qdrant, Weaviate, Elasticsearch v7 and v8, OpenSearch, the Neo4j driver, and AWS/Alibaba/Tencent/Volcengine/Huawei/Kingsoft storage SDKs.
- `mcp-go` for MCP support and gojieba for tokenization; a Wails dependency is also present.
- Deployment via Docker Compose (profiles for neo4j, minio, langfuse, and full) and Kubernetes/Helm. Version 0.7.0.

## When to reach for it

- You need a self-hosted RAG plus agent knowledge base with full data sovereignty.
- You want to swap LLMs, vector databases, and object storage without rebuilding your stack.
- You need multi-tenant workspaces with RBAC and audit logging.
- You want to serve knowledge Q&A through IM channels or embed it into external websites.

## When *not* to reach for it

- You want a lightweight, single-file library — this is a full multi-service platform requiring Docker or Kubernetes, Redis, and a vector database.
- You do not need document ingestion or RAG — a plain LLM API wrapper is simpler.
- You lack the infrastructure to operate the required services.
- Your team reads only English — some deeper guides (for example the development guide and RBAC documentation) are Chinese-only.

## Maturity signal

Backed by Tencent, created in July 2025, WeKnora reached roughly 18,700 stars within a year, was last pushed in July 2026, and is not archived, with a fast release cadence that moved from v0.2 to v0.7 while adding substantial features at each step. It remains pre-1.0 (v0.7.0), so ongoing change should be expected, and the 434 open issues are consistent with heavy adoption. The README labels the license MIT, but GitHub's detected SPDX is NOASSERTION, so the `LICENSE` file is worth verifying before reuse.

## Alternatives

- RAGFlow — use it when you want a RAG engine centered on deep document parsing.
- Dify — use it when you want a broader visual LLM-app builder rather than a knowledge-base-first platform.
- LangChain or LlamaIndex — use them when you want to assemble a custom RAG pipeline as a library instead of deploying a full platform.

## Notes

- The README states the license is MIT while GitHub reports NOASSERTION, a mismatch worth confirming.
- Langfuse is the sole tracing backend; the changelog notes Jaeger was removed in v0.6.2.
- WeKnora is described as the core framework behind Tencent's WeChat Dialog Open Platform.
- A Wails dependency in `go.mod` suggests a desktop build path alongside the web UI.

## Tags

golang, rag, llm, knowledge-base, vector-search, semantic-search, agent, self-hosted, multi-tenant, chatbot, reranking
