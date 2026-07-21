# khoj-ai/khoj

A self-hostable personal AI application that answers questions from the web or your own documents, using local or hosted LLMs.

## What it is

Khoj is a Python application that acts as a "second brain": it lets you chat with a language model that can retrieve answers from the internet and from your own files. It targets individuals who want a private, self-hosted AI assistant, as well as teams that need it as a cloud or on-premises service. The project spans on-device use up to cloud-scale deployment, and connects to a range of local and online models (llama3, qwen, gemma, mistral, gpt, claude, gemini, deepseek).

## Key features

- Chat with local or online LLMs, including offline models via llama.cpp/Ollama-style setups and hosted providers (OpenAI, Anthropic, Google).
- Retrieval over your own documents, including images, PDF, Markdown, org-mode, Word, and Notion files.
- Semantic search over indexed documents using sentence-transformer embeddings and pgvector.
- Custom agents with their own knowledge, persona, chat model, and tools.
- Scheduled automations that deliver newsletters and notifications (backed by APScheduler and cron scheduling).
- Multiple client surfaces: web, Obsidian, Emacs, desktop, phone, and WhatsApp.
- Additional capabilities noted in the README: image generation, voice/text-to-speech, and code execution (the manifest includes an e2b code interpreter dependency).

## Tech stack

- Python, required `>=3.10, <3.13`.
- Web/API layer: FastAPI (`>=0.110.0`) served with Uvicorn/Gunicorn; websockets for streaming.
- Application/data layer: Django (`5.1.15`) with django-unfold admin, PostgreSQL via psycopg2, and pgvector (`0.2.4`) for vector search; local Postgres via pgserver for the `local` extra.
- ML/embeddings: sentence-transformers (`3.4.1`), transformers (`>=4.53.0`), torch (`2.6.0`), tiktoken.
- LLM SDKs: openai (`>=2.0.0, <3.0.0`), anthropic (`0.75.0`), google-genai (`1.52.0`), plus the `mcp` package (`>=1.23.0`).
- Document parsing: PyMuPDF, docx2txt, markdownify, markdown-it-py, BeautifulSoup, lxml, rapidocr-onnxruntime; openai-whisper for speech-to-text.
- Scheduling/retrieval: APScheduler with django_apscheduler, langchain-text-splitters and langchain-community for chunking/retrieval helpers.
- Auth and messaging: Authlib, phonenumbers; optional prod extras add Stripe, Twilio, and boto3.
- Packaging: hatchling with hatch-vcs; tooling includes mypy, ruff, and pytest (pytest-django, pytest-asyncio).
- Clients live in the repo tree as well, including an Android app (Java/Gradle) under `src/interface`.

## When to reach for it

- You want a private, self-hosted AI assistant that keeps your documents on your own infrastructure.
- You need question answering grounded in a personal corpus spanning mixed formats (PDF, Markdown, org-mode, Word, Notion, images).
- You want to use offline/local models rather than sending data to a hosted provider, or mix local and hosted models.
- You want the same assistant reachable from several clients (web, Obsidian, Emacs, desktop, phone, WhatsApp).
- You want to build custom agents and schedule recurring research or notifications.

## When *not* to reach for it

- You need a lightweight, embeddable library rather than a full application. Khoj is a Django + FastAPI service with PostgreSQL and pgvector as core dependencies, so it carries real deployment weight.
- You cannot run or manage PostgreSQL/pgvector and a Python 3.10–3.12 environment with a torch-based ML stack.
- AGPL-3.0 licensing is incompatible with your distribution or product model.
- You only need a single-format, small-scale search and don't want the overhead of embeddings, a database, and multiple model integrations.

## Maturity signal

Khoj is actively maintained. It carries roughly 35.9k stars, was created in 2021, and had a recent push (mid-2026 in the provided facts), with the pyproject classifiers marking it "Production/Stable". The dependency list pins recent versions across the ML and web stack and integrates current model SDKs, which indicates ongoing upkeep rather than long-tail maintenance. A relatively low open-issue count (123) against the star count and an active contributor base and Discord suggest a project with steady development and community support.

## Alternatives

- Open WebUI — reach for it when you primarily want a polished chat front end over Ollama/OpenAI-compatible models and don't need Khoj's document-corpus and multi-client focus.
- AnythingLLM — reach for it when you want a document-chat/RAG desktop or server app with a simpler stack and less emphasis on editor/messaging clients.
- PrivateGPT — reach for it when your main goal is fully local document Q&A and you prefer a smaller, more focused codebase over Khoj's broader feature surface.

## Notes

- Although the README frames Khoj as a personal/local AI, the codebase is built on Django and PostgreSQL with pgvector, closer to a server application than a desktop tool.
- The repository bundles native clients directly, including an Android app (Java/Gradle) alongside the Python backend.
- Optional `prod` extras pull in Stripe, Twilio, and boto3, reflecting the hosted cloud/enterprise offering described in the README rather than the self-hosted path.
- The license is AGPL-3.0, which is notable for anyone considering deploying it as a network service or building a derived product.

## Tags

python, django, fastapi, llm, rag, semantic-search, self-hosted, ai-assistant, vector-search, pgvector, agents, machine-learning
