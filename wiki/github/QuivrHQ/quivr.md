# QuivrHQ/quivr

An opinionated Python RAG core you install as a library so you can add retrieval-augmented generation to your product instead of building the RAG yourself.

## What it is

Quivr is the RAG "brain" behind Quivr.com, packaged as `quivr-core` so developers can ingest files and ask questions from a few lines of Python. It is deliberately opinionated: it provides a working retrieval-augmented-generation setup out of the box so you can focus on your product rather than assembling retrieval, chunking, and generation from scratch. It is aimed at developers integrating GenAI into existing apps who want customization (internet search, tools, different retrieval strategies) without committing to a single LLM or vector store.

## Key features

- A `Brain` abstraction that ingests files and answers questions in roughly five lines of code (`Brain.from_files(...)`, `brain.ask(...)`).
- Model-agnostic LLM support (OpenAI, Anthropic, Mistral, Gemma, and local models via Ollama) and multiple vector stores (PGVector, Faiss).
- Broad file support (PDF, TXT, Markdown, and more) with pluggable processors, including custom parsers.
- YAML-configured retrieval workflows built on a graph of nodes (filter history, rewrite, retrieve, generate), with a configurable reranker (for example Cohere) and history/token limits.
- Integration with Megaparse for file ingestion.

## Tech stack

- Python (3.10+ per the README and `.python-version`); the RAG lives in `core/quivr_core` with its package config in `core/pyproject.toml`.
- LangGraph-based RAG implementation (`quivr_rag_langgraph.py`) alongside a classic `quivr_rag.py`.
- Vector stores: PGVector and Faiss; reranking via a configurable supplier such as Cohere.
- Document processing via multiple implementations (default, Megaparse, simple text, Tika), with tests covering PDF, DOCX, EPUB, and ODT.
- Local storage and pluggable storage backends; configuration loaded from YAML (`RetrievalConfig.from_yaml`).
- Docs built with MkDocs; tests via pytest/tox with lockfiles (`requirements.lock`). No recognized root manifest at the repo top — the installable package is under `core/`.

## When to reach for it

- You want to embed RAG into an existing product by installing `quivr-core` rather than standing up a separate service.
- You need to stay flexible on LLM provider and vector store, including running local models through Ollama.
- You want retrieval behavior expressed as an editable YAML workflow you can iterate on quickly.
- You are ingesting mixed document types (PDF, text, Markdown, and others) and may want to add custom parsers.

## When *not* to reach for it

- You want a full end-user application with a hosted UI out of the box; this repository is the core library, not the product front end.
- You need a broad, general-purpose agent framework beyond retrieval-augmented generation.
- You require the most granular, low-level control over every indexing and retrieval component and would rather not adopt an opinionated default pipeline.
- You need guarantees of frequent upstream updates, given the pace signal below.

## Maturity signal

Quivr has been developed since May 2023 and reached large star and fork counts, reflecting sustained popularity and a real community. However, the last push recorded here is from July 2025 — roughly a year before this writing — so despite not being archived, the core repository appears to have slowed to long-tail maintenance rather than active daily development; the modest open-issue count fits a project that is stable but not rapidly moving. The README states the license is Apache 2.0 (the platform reports it as NOASSERTION). Treat it as a mature, widely used library that may not be receiving frequent updates; verify recency before depending on it for new work.

## Alternatives

- LlamaIndex — reach for it when you want fine-grained control over indexing and retrieval components rather than an opinionated default RAG.
- LangChain — use when you need a broad, general LLM/agent framework beyond retrieval-augmented generation.
- Haystack — use for production NLP/RAG pipelines with strong document-store integrations.
- RAGFlow — use when you want a full self-hosted RAG application with a user interface rather than an embeddable Python library.

## Notes

- The quickstart's sample text deliberately asserts a false fact ("Gold is a liquid of blue-like colour") to demonstrate that answers are grounded in the ingested document rather than the model's prior knowledge.
- Megaparse is a sibling project by the same organization, and the README frames Quivr as pairing with it for ingestion.

## Tags

python, rag, llm, library, vector-search, langgraph, generative-ai, chatbot
