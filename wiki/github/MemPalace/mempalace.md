# MemPalace/mempalace

A local-first AI memory system that stores conversation and project history as verbatim text and retrieves it with semantic search, with no API calls required for its core path.

## What it is

MemPalace mines projects and conversations into a searchable store and retrieves them with semantic search, without summarizing, extracting, or paraphrasing the original content. It is aimed at developers using AI coding agents — Claude Code, Codex CLI, Cursor, Gemini CLI — who want persistent, searchable memory that stays on their own machine. The index is structured: people and projects become "wings," topics become "rooms," and original content lives in "drawers," so searches can be scoped rather than run against a flat corpus. The retrieval layer is pluggable, defined by a backend interface, with ChromaDB as the default.

## Key features

- Verbatim storage with semantic retrieval — no summarization, extraction, or paraphrasing of stored content.
- Pluggable storage backends: `chroma` (default), `sqlite_exact`, `milvus`, `qdrant`, and `pgvector`, selectable via `--backend`, `MEMPALACE_BACKEND`, or `config.json`.
- CLI workflow (`mempalace mine`, `search`, `wake-up`, `sweep`) for mining project files and Claude Code session transcripts.
- MCP server exposing 36 tools covering palace reads/writes, knowledge-graph operations, cross-wing navigation, drawer management, and agent diaries.
- A temporal entity-relationship knowledge graph with validity windows (add, query, invalidate, timeline) backed by local SQLite.
- Auto-save hooks for Claude Code, Codex CLI, and Cursor IDE that persist periodically and before context compaction.

## Tech stack

- Python, requires-python `>=3.9` (classifiers list through 3.14).
- Core dependencies: `chromadb>=1.5.4,<2`, `pyyaml>=6.0,<7`, `huggingface_hub>=0.20`, `tokenizers>=0.15`, `numpy>=1.24`, `python-dateutil>=2.8`, and `tomli` on Python < 3.11.
- Embedding models: `embeddinggemma-300m` (multilingual, default for new installs) or `all-MiniLM-L6-v2` (English-only, ~30 MB); ~300 MB disk for the default model.
- Optional extras: `milvus` (pymilvus), `pgvector` (psycopg), `extract` (striprtf, markitdown for PDF/DOCX/PPTX/XLSX/EPUB), and `gpu`/`dml`/`coreml` ONNX runtime variants.
- Build backend `hatchling`; dev tooling includes pytest, pytest-cov, ruff `0.15.20`, mypy, and hypothesis.
- Docker images for CPU and GPU (CUDA) that run the CLI or MCP server, persisting under `/data`.

## When to reach for it

- You want a local-first memory layer for AI agents where data does not leave your machine unless you opt in.
- You need exact, verbatim recall of past conversations rather than model-generated summaries.
- You are mining Claude Code, Codex CLI, or Cursor sessions and want them searchable and scoped by project or topic.
- You want to wire memory into an MCP-compatible client through a stdio server.
- You want to run entirely without an API key for the core retrieval path.

## When *not* to reach for it

- You want a hosted, managed memory service rather than something you install and run yourself.
- Your workflow depends on summarized or extracted memories; MemPalace deliberately stores raw text.
- You cannot spare roughly 300 MB of disk for the default embedding model.
- You need a large-scale distributed vector store as a turnkey SaaS; server backends (Milvus, Qdrant, pgvector) are opt-in and self-configured.

## Maturity signal

The project reports itself as version 3.6.0 with a "Development Status :: 4 - Beta" classifier, is MIT-licensed, and shows recent activity (last push in mid-2026 against an early-2026 creation date). The combination reads as young but under active development — the feature surface (multiple backends, an MCP server, a knowledge graph, auto-save hooks across three agents) is broad for its age, and the maintainers explicitly track corrections and public notices in a history file. Treat it as an actively developed beta rather than a settled 1.x release.

## Alternatives

- Mem0 — use it instead when you want a managed memory layer that extracts and summarizes rather than storing verbatim.
- Zep — use it instead when you need a server-based memory service with its own end-to-end QA metrics.
- Supermemory — use it instead when you prefer a hosted product over running and configuring a local store.

## Notes

- The README carries a prominent warning about impostor sites distributing malware and names its only official sources; that is unusual to see foregrounded in a project README.
- The benchmarking stance is deliberately conservative: it publishes a "raw" 96.6% R@5 figure requiring no LLM, headlines a held-out 98.4% as the honest generalizable number, and declines side-by-side comparisons against Mem0, Mastra, Hindsight, Supermemory, or Zep on the grounds that mismatched metrics are not an honest comparison.
- The LLM rerank path is described as model-agnostic and has been reproduced with Claude Haiku, Claude Sonnet, and a model via Ollama Cloud.

## Tags

python, memory, llm, rag, vector-database, chromadb, mcp, cli, embeddings, semantic-search, local-first, knowledge-graph
