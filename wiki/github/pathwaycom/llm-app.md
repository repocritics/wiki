# pathwaycom/llm-app

Ready-to-run templates for building RAG and enterprise-search pipelines that stay in sync with live data sources.

## What it is

This repository provides deployable LLM application templates built on the Pathway Live Data Framework, aimed at teams putting RAG and AI enterprise search into production. The apps connect to your data sources, keep an in-memory index synced as data changes, and expose an HTTP API you can point a frontend at. They are meant to be testable on a local machine and deployable to cloud providers or on-premises without assembling a separate vector database, cache, and API framework.

## Key features

- A set of ready-to-run templates including question-answering RAG, live document indexing, multimodal RAG with GPT-4o, unstructured-to-SQL, adaptive RAG, private local RAG (Mistral + Ollama), slides search, and video RAG with TwelveLabs.
- Live synchronization of additions, deletions, and updates from file systems, Google Drive, Sharepoint, S3, Kafka, PostgreSQL, and real-time data APIs.
- Built-in in-memory indexing supporting vector search, hybrid search, and full-text search, with caching and no external vector database required.
- Apps run as Docker containers exposing an HTTP API, with an optional Streamlit UI for quick demos.
- Adaptive RAG, a Pathway technique, is described as reducing RAG token cost up to 4x while maintaining accuracy.

## Tech stack

- Python `>=3.10,<3.13`, managed with Poetry (`package-mode = false`).
- Depends on the Pathway Live Data Framework (`pathway = "^0.12.0"`), a standalone Python library with a Rust engine built in.
- Vector indexing uses the usearch library; hybrid full-text indexing uses Tantivy.
- Templates are primarily Jupyter Notebook and Python; deployment is via Docker with an HTTP API.
- Linting/dev group: black `^24.2`, isort, mypy `~1.10`, flake8 `~7.0`, pytest `^8.0`.

## When to reach for it

- Deploying a RAG or enterprise-search pipeline that must stay current with changing document sources.
- Wanting a single unified stack instead of integrating a separate vector DB, cache, and API framework.
- Extracting from unstructured documents including charts and tables, using multimodal parsing with GPT-4o.
- Running a fully private, local RAG pipeline with Mistral and Ollama.

## When *not* to reach for it

- You already run and want to keep a dedicated vector database such as Pinecone, Weaviate, or Qdrant; this framework replaces that layer.
- Your corpus needs distributed or persistent vector storage beyond what in-memory indexing provides.
- You do not need live sync and a lighter one-off RAG library would suffice.
- Your stack is not Python-based.

## Maturity signal

With roughly 59k stars, creation back in 2023, a recent last push, and an MIT license, this is a popular, company-backed project maintained by Pathway. The `pyproject.toml` still marks it "3 - Alpha" at version 0.3.6, so the packaging is versioned conservatively even though adoption and upkeep are strong; treat it as actively maintained with an evolving API.

## Alternatives

- **LangChain / LlamaIndex** — use them when you want a general framework and prefer to bring your own vector store; the README notes Pathway can act as a retriever backend for both.
- **A dedicated vector DB (Pinecone / Weaviate / Qdrant) plus FastAPI** — use this when you want to own and tune each layer separately; the README frames Pathway as replacing that trio.

## Notes

Pathway is unusual for a Python RAG stack in that its engine is written in Rust, and the framework deliberately collapses the common vector-database, cache, and API-framework trio into one in-memory system. The video RAG template is also notable: it uses TwelveLabs Pegasus to turn video into text descriptions and Marengo embeddings to index it. GitHub reports the repository's primary language as Jupyter Notebook, reflecting the notebook-heavy templates rather than the underlying library.

## Tags

python, rag, llm, vector-database, machine-learning, real-time, retrieval-augmented-generation, docker, enterprise-search, framework
