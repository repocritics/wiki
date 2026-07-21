# chatchat-space/Langchain-Chatchat

An offline-deployable RAG and Agent application that runs question-answering over a local knowledge base using open-source LLMs.

## What it is

Langchain-Chatchat (formerly Langchain-ChatGLM) is a Python application for building local knowledge-base question answering on top of the LangChain framework and open-source language models such as ChatGLM, Qwen, and Llama. It targets teams that want a RAG and Agent system that can run fully offline with open-source models and vector databases, while still allowing calls to hosted APIs like OpenAI. The pipeline loads documents, splits and embeds text, retrieves the top-k chunks most similar to a question, and passes them as context to an LLM to generate an answer. It is packaged for private, self-hosted deployment.

## Key features

- Retrieval-augmented chat over a local knowledge base, plus plain LLM chat, search-engine chat, file chat (unified File RAG with BM25 + KNN retrieval), and database chat.
- Agent workflow optimized for ChatGLM3 and Qwen, with a tool registry covering internet search, local knowledge-base search, ArXiv, Wolfram, text-to-SQL, text-to-image, weather, and more.
- Multimodal image chat (recommended with qwen-vl-chat) and text-to-image generation.
- Pluggable model inference via external frameworks — Xinference, Ollama, LocalAI, FastChat — and hosted APIs through One API, all aligned to the OpenAI SDK.
- FAISS vector store by default, with configuration for other vector database types; YAML config files that the server reloads without a restart.
- Streamlit-based WebUI and a FastAPI-based API server, installed and run through a `chatchat` CLI (`chatchat init`, `chatchat kb -r`, `chatchat start -a`).

## Tech stack

- Python, supported on 3.8–3.11 (`pyproject.toml` pins `>=3.8.1,<3.12,!=3.9.7`).
- LangChain as the application framework.
- FastAPI for the API server and Streamlit for the WebUI.
- FAISS as the default vector store (Milvus and other backends referenced in project topics).
- Poetry-based packaging; distributed on PyPI as `langchain-chatchat` (with an optional `[xinference]` extra).
- Runs on CPU, GPU, NPU, and MPS hardware; tested on Windows, macOS, and Linux.

## When to reach for it

- You need a knowledge-base chatbot that can run entirely offline with open-source LLMs and embedding models.
- You want private, self-hosted deployment where documents never leave your infrastructure.
- Your models are served through Xinference, Ollama, LocalAI, or FastChat and you want a ready-made RAG/Agent layer on top.
- You want a working WebUI plus REST API out of the box rather than assembling retrieval, chat, and agent tooling yourself.

## When *not* to reach for it

- You want a single `pip install` with no external model-serving setup; from 0.3.0 you must run a separate inference framework and load models before starting the app.
- You need long-term API stability; the README notes the 0.3.x architecture changed substantially and does not guarantee compatibility with 0.2.x deployments.
- You are building a lightweight embedded retrieval component rather than a full self-hosted server with WebUI and knowledge-base management.
- You require binding to a public network address by default — the default `DEFAULT_BIND_HOST` is `127.0.0.1` and must be changed to expose the service.

## Maturity signal

With over 38,000 stars and a history dating to March 2023, this is one of the most widely known Chinese-language local-RAG projects, and it remains under the Apache-2.0 license with an open-issue count (26) that is low relative to its size. The most recent push was November 2025, and the published milestones trail off after the 0.3.0 architecture rewrite in mid-2024, which suggests a mature project that is still maintained but developing at a slower pace than during its 2023–2024 peak. Note a licensing discrepancy worth checking before adoption: the README and repository metadata state Apache-2.0, while the root `pyproject.toml` declares `license = "MIT"`.

## Alternatives

- Dify — use it instead when you want a hosted or self-hosted LLM app platform with a visual builder rather than a CLI-driven Python deployment.
- RAGFlow — use it instead when deep document parsing and layout-aware ingestion are your primary concern.
- AnythingLLM — use it instead when you want a simpler desktop/server knowledge-base chat app with less model-framework setup.
- LlamaIndex — use it instead when you want a library to compose your own retrieval pipeline rather than a packaged application.

## Notes

From version 0.3.0, the project stopped loading models from local paths directly and instead delegates all model loading to external inference frameworks kept in separate Python environments to avoid dependency conflicts. Configuration moved to editable local YAML files that the server re-reads automatically, so most settings changes do not require a restart.

## Tags

python, rag, retrieval-augmented-generation, llm, langchain, knowledge-base, chatbot, agent, self-hosted, fastapi, streamlit, vector-database
