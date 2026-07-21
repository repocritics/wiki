# infiniflow/ragflow

An open-source Retrieval-Augmented Generation engine that combines deep document understanding with agent capabilities to build a context layer for LLMs.

## What it is

RAGFlow is a self-hostable RAG engine aimed at developers and enterprises that need question-answering over their own documents. It extracts knowledge from unstructured data in complicated formats, chunks it using explainable templates, and pairs retrieval with re-ranking so that answers come back with traceable citations. Beyond plain RAG, it ships pre-built agent templates and an orchestration layer, positioning itself as a "context engine" that turns complex data into production AI systems.

## Key features

- Deep document understanding for knowledge extraction from unstructured data with complicated formats, including Word, slides, Excel, txt, images, scanned copies, structured data, and web pages.
- Template-based chunking that is described as intelligent and explainable, with multiple template options.
- Grounded citations with visualization of text chunking for human intervention and traceable references to reduce hallucinations.
- Configurable LLMs and embedding models, with multiple recall paired with fused re-ranking.
- Agentic workflow support, MCP, and pre-built agent templates (for example deep research, text2sql, trip planner, and stock market research).
- A Python/JavaScript code executor component for agents, run in a gVisor-based sandbox when enabled.

## Tech stack

- Go (`go 1.26.4`) as the primary repository language, using Gin (`gin-gonic/gin v1.12.0`), GORM (`gorm.io/gorm v1.25.7`), the CloudWeGo eino framework (`cloudwego/eino v0.9.12`), and OpenTelemetry.
- Python backend components (`requires-python >=3.13,<3.14`) built on Flask/Quart, peewee, and langgraph (`==1.2.0`).
- Document/full-text and vector storage via Elasticsearch by default (`go-elasticsearch/v8`, `elasticsearch-dsl==8.12.0`), with an option to switch to Infinity (`infinity-sdk==0.7.2`) or OpenSearch.
- Supporting services: MySQL, MinIO object storage (`minio-go/v7`, `minio==7.2.4`), and Redis/Valkey.
- Broad LLM and provider SDK coverage including anthropic (`==0.76.0`), cohere, mistralai, groq, ollama, zhipuai, dashscope/qianfan, voyageai, google-genai, and litellm (`==1.82.5`); MCP support via `mcp>=1.19.0`.
- Delivered primarily as Docker images (Docker >= 24.0.0, Docker Compose >= v2.26.1); development uses `uv` for Python dependency management.
- Licensed Apache-2.0.

## When to reach for it

- You need a self-hosted RAG system over heterogeneous document formats rather than a hosted-only or single-format tool.
- Citation traceability and human-inspectable chunking matter for your answers.
- You want to combine retrieval with agent workflows, MCP tools, or code-executor steps in one platform.
- You want to plug in your own choice of LLM and embedding providers instead of being locked to one.
- You prefer deploying via Docker Compose and can meet the resource baseline (CPU >= 4 cores, RAM >= 16 GB, Disk >= 50 GB).

## When *not* to reach for it

- You are on ARM64 and want turnkey images; all official Docker images are built for x86, and ARM64 requires building your own (switching to Infinity on Linux/arm64 is also not officially supported).
- You want a lightweight embeddable library; RAGFlow is a multi-service system requiring Elasticsearch/Infinity, MySQL, MinIO, and Redis.
- Your environment cannot meet the stated minimums or run Docker/Docker Compose.
- You need the sandboxed code executor without installing gVisor, which is a listed prerequisite for that feature.

## Maturity signal

RAGFlow is actively maintained and widely adopted: about 85,600 stars, an Apache-2.0 license, and a last push in July 2026 for a project created in December 2023. Frequent dated updates through 2026 (multi-channel chat, DeepSeek v4, agent memory, MinerU/Docling parsing) and versioned Docker releases (v0.26.4) indicate ongoing, fast-moving development rather than maintenance mode. The roughly 2,100 open issues are consistent with a large, active user base rather than neglect.

## Alternatives

- Haystack — reach for it when you want a Python-native RAG/LLM framework you assemble in code rather than a full self-hosted server.
- LlamaIndex — reach for it when you primarily need a data framework and indexing library to embed in your own application.
- Dify — reach for it when your emphasis is an LLM app/agent orchestration platform with a broad app builder rather than deep document parsing.
- Infinity — reach for it directly when you only need the vector/full-text database layer; RAGFlow can use it as its document engine.

## Notes

- RAGFlow can swap its document engine from Elasticsearch to Infinity, a sibling project from the same organization (infiniflow/infinity).
- The repository carries agent-oriented tooling for its own development, including `.agents/`, `AGENTS.md`, `CLAUDE.md`, and `.github/copilot-instructions.md`, plus a documented OpenClaw skill for accessing RAGFlow datasets.
- Despite Go being the primary language, a large Python surface remains; the pyproject pins many dependencies with explicit comments explaining CVE mitigations and broken-release avoidance (for example pinning litellm to 1.82.5 and constraining pyasn1, trio, and urllib3).

## Tags

go, python, rag, retrieval-augmented-generation, llm, machine-learning, agent, self-hosted, document-parsing, vector-database, docker, mcp
