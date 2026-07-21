# HKUDS/LightRAG

A lightweight knowledge-graph RAG framework that combines graph indexing and vector embeddings for retrieval-augmented generation, positioned as an efficient alternative to Microsoft GraphRAG.

## What it is

LightRAG is a Python framework and server for retrieval-augmented generation that manages both a knowledge graph and vector embeddings in a dual-layer architecture. It targets developers building RAG systems over large or evolving document sets who want graph-structured context and incremental updates without the heavy computation and cost of community-report or multi-hop approaches. The project accompanies an EMNLP 2025 paper and ships as both an importable library and an API/WebUI server. It supports open-source LLMs down to around 30B parameters while aiming to keep retrieval quality high.

## Key features

- Dual-level retrieval with five query modes — `local`, `global`, `hybrid`, `naive`, and `mix` (the default) — spanning specific-entity lookups and cross-document reasoning.
- Incremental knowledge-base updates via set merging, plus document deletion with automatic knowledge-graph regeneration, avoiding full index rebuilds.
- Role-specific LLM configuration across four roles (EXTRACT, QUERY, KEYWORD, VLM) so different models can be assigned by cost and capability.
- Multimodal document processing (from v1.5) through MinerU, Docling, and a native parser for text, tables, formulas, and images, with the RAG-Anything project merged in.
- Optional reranking (default query mode) and four selectable chunking strategies (Fix, Recursive, Vector, Paragraph).
- Pluggable backends for four storage types (KV, vector, graph, doc-status), a REST API with a web UI, and RAGAS evaluation plus Langfuse tracing integrations.

## Tech stack

- Python, requiring `>=3.10`; recommended package management via `uv`.
- Core dependencies include `networkx`, `nano-vectordb`, `numpy`, `pandas`, `tiktoken`, `pydantic`, and `tenacity`.
- API server built on FastAPI (`fastapi>=0.108`) with `uvicorn`/`gunicorn`, plus security-pinned floors for `starlette`, `python-multipart`, and `cryptography`.
- Storage backends (optional extras) span PostgreSQL/`pgvector`, MongoDB, OpenSearch, Milvus, Qdrant, Neo4j, Memgraph, Redis, and Faiss.
- LLM/provider integrations (optional extras) include OpenAI, Anthropic, Ollama, Zhipu, AWS via `aioboto3`, VoyageAI, Google GenAI, and LlamaIndex.
- Web UI built and served from `lightrag_webui` using `bun`; ships Docker Compose, Podman, and Kubernetes/Helm deployment paths.

## When to reach for it

- You want graph-based RAG that captures relationships between entities for global or cross-document questions.
- Your knowledge base changes frequently and you need low-cost incremental inserts and deletions rather than periodic full re-indexing.
- You are running open-source or self-hosted LLMs and embedding models and want to keep LLM call counts and latency down.
- You need a single project that spans ingestion, multimodal parsing, retrieval, a REST API, and a web UI.

## When *not* to reach for it

- You need a production system out of the box with the defaults; the built-in file-persisted, in-memory storages are described as development/debugging only.
- You expect to swap embedding models freely; embeddings must be fixed before indexing, and changing them requires re-embedding all content with no built-in re-embedding tool.
- Your use case is simple chunk-and-vector retrieval; the knowledge-graph extraction step raises LLM capability requirements and indexing cost.
- You cannot tolerate the operational overhead of running external databases (Postgres, Neo4j, Milvus, etc.) for a production deployment.

## Maturity signal

LightRAG has nearly 38,000 stars since its October 2024 creation and a most-recent push in July 2026, marking it as actively and rapidly developed, with the packaging classifiers still labeling it Development Status 4 (Beta). The steady stream of dated feature entries through 2026 (OpenSearch storage, a setup wizard, role-specific LLM config, RagAnything merge) and its EMNLP 2025 publication indicate a research-backed project under continuous expansion; the 217 open issues reflect wide adoption rather than neglect. It is MIT licensed.

## Alternatives

- Microsoft GraphRAG — the README positions LightRAG as an efficient alternative; use GraphRAG when you specifically want its community-report and multi-hop approach.
- LlamaIndex — use it instead when you want a broad, general-purpose RAG toolkit rather than a graph-first framework.
- RAGFlow — use it instead when layout-aware document parsing and an end-to-end product UI matter more than graph retrieval.
- MiniRAG — from the same team, use it instead when you want RAG simplified for small models.

## Notes

The framework requires four distinct storage roles and can back all of them with a single database (PostgreSQL, MongoDB, or OpenSearch) or mix specialized ones. It documents unusually specific operational guidance, including that the effective LLM execution timeout is twice the configured `*_LLM_TIMEOUT` value, and that reference/bibliography sections can flood entity extraction — mitigated by dropping references under the paragraph (`P`) chunking strategy. The server binds to `0.0.0.0` by default with all endpoints public unless authentication is configured.

## Tags

python, rag, retrieval-augmented-generation, knowledge-graph, graphrag, llm, vector-database, nlp, library, framework
