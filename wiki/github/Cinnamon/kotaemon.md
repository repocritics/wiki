# Cinnamon/kotaemon

An open-source, self-hosted RAG web UI for chatting with your documents, aimed at both end users doing document QA and developers building their own retrieval pipelines.

## What it is

Kotaemon is a retrieval-augmented-generation application and framework for asking questions over your own documents. It serves two audiences: end users who want a clean question-answering interface over their files, and developers who want to build and customize a RAG pipeline with a working UI already attached. The UI is built with Gradio, and the codebase is split into a reusable library (`kotaemon`) and the application layer (`ktem`). It supports both hosted LLM providers and local models.

## Key features

- Self-hosted document QA web UI with multi-user login and private/public collections for organizing and sharing files and chats.
- Support for both API providers (OpenAI, Azure OpenAI, Cohere, Groq) and local LLMs via `ollama` and `llama-cpp-python`.
- A hybrid RAG pipeline combining full-text and vector retrieval with re-ranking.
- Multi-modal QA over documents with figures and tables, using selectable document parsers.
- Advanced citations shown in an in-browser PDF viewer with highlights and relevance scores, plus warnings when retrieved passages are low-relevance.
- Complex reasoning via question decomposition and agent-based methods (ReAct, ReWOO), and an extensible indexing layer with GraphRAG provided as an example.

## Tech stack

- Python `>=3.10`; Apache-2.0 licensed.
- UI built on Gradio; project structured as a `uv` workspace with two packages under `libs/` (`kotaemon` and `ktem`), using `setuptools` with `setuptools-git-versioning`.
- Configurable document stores (Elasticsearch, LanceDB, SimpleFileDocumentStore) and vector stores (ChromaDB, LanceDB, InMemory, Milvus, Qdrant), set via `flowsettings.py`.
- Optional multimodal parsing/OCR through Azure Document Intelligence, Adobe PDF Extract, Docling, PaddleOCR, and Unstructured for non-PDF file types.
- GraphRAG indexing options: official MS GraphRAG, NanoGraphRAG, and LightRAG.
- Distributed as Docker images on GHCR (`lite`, `full`, and `ollama` variants) for `linux/amd64` and `linux/arm64`, plus a `pip install -e` path.

## When to reach for it

- You want a self-hosted UI for document QA that non-developers can use out of the box.
- You need multi-user access with organized private/public document collections.
- Verifiable answers matter and you want inline citations with a PDF viewer and relevance scores.
- You are a developer who wants a customizable RAG pipeline with a ready-made UI to iterate against.
- You want local, private RAG using Ollama or GGUF models.

## When *not* to reach for it

- You need a headless RAG library only; the Gradio UI is central to the project rather than optional.
- You are targeting very high-throughput or large-scale production serving, where a Gradio-based app may not be the right layer.
- You want a fully managed cloud service rather than something you deploy and operate yourself.
- Your use case goes beyond document question-answering into workflows the pipeline is not designed for.

## Maturity signal

Created in March 2024, kotaemon is one of the more established projects in this set, with roughly 25.6k stars and a last push about a week before this snapshot, so it remains actively maintained rather than dormant. It is Apache-2.0, not archived, and backed by named maintainers at Cinnamon with a support contact. Its open-issue count (around 236) is moderate and consistent with a widely used application that draws steady user reports. Overall this reads as a mature, actively developed tool.

## Alternatives

- Use **AnythingLLM** or **Open WebUI** instead when you want a broader chat application and model-management experience rather than a citation-focused document-QA UI.
- Use **RAGFlow** instead when deep document layout parsing and ingestion are your priority.
- Use **LlamaIndex** or **Haystack** instead when you want a library to build a RAG system from scratch without a bundled UI.
- Use **Onyx (formerly Danswer)** instead when you need enterprise search across many connected data sources rather than QA over uploaded files.

## Notes

- The project is deliberately layered for three audiences — end users, developers who `import kotaemon`, and contributors — and ships both a library and an app to serve them.
- GraphRAG support is pluggable across three implementations, but the README notes official MS GraphRAG indexing works only with OpenAI or Ollama and recommends NanoGraphRAG for most users.
- Enabling the in-browser PDF viewer requires manually downloading and extracting a specific `pdf.js` distribution into the assets directory.

## Tags

python, rag, llm, chatbot, gradio, document-qa, information-retrieval, graphrag, web-ui, self-hosted
