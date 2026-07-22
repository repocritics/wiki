# HKUDS/DeepTutor

An agent-native, self-hosted learning workspace that unifies tutoring, problem solving, quizzing, research, and visualization on a single agent loop.

## What it is

DeepTutor is a personalized tutoring application where chat, quiz generation, deep research, visualization, problem solving, and mastery practice all run on the same agent engine, so switching modes changes the objective rather than the tool. It connects knowledge bases, books, notebooks, question banks, personas, and a layered memory system so learner context carries across every workflow. It targets learners and educators who want a self-hosted, extensible study environment backed by their own LLM providers.

## Key features

- One agent runtime shared across Chat, Quiz, Research, Visualize, Solve, and Mastery Path, with context that moves with the learner.
- Multi-engine knowledge bases with versioned RAG across LlamaIndex, PageIndex, GraphRAG, LightRAG, or a linked Obsidian vault, plus pluggable document parsing.
- Subagents and Partners: consult a live Claude Code, Codex, or a configured Partner mid-turn, and run persistent IM companions across many chat channels.
- Inspectable three-layer memory (L1 traces, L2 summaries, L3 synthesis) with a Memory Graph tracing claims back to evidence.
- Extensible tools and installable community skills from EduHub, plus image, video, and voice generation models.
- Four installation paths (PyPI, source, Docker, CLI-only) sharing one workspace layout, with a `deeptutor` CLI.

## Tech stack

- Backend: Python 3.11+, built on FastAPI and uvicorn, with a Typer-based CLI, WebSockets, PocketBase, and SQLite (`aiosqlite`).
- Frontend: Next.js 16 and React 19, requiring a Node.js runtime (20+ for the packaged server, 22 LTS for source development).
- LLM and RAG: native OpenAI and Anthropic SDKs, LlamaIndex `>=0.14.12`, FAISS (`faiss-cpu`), PyMuPDF, and BM25 retrieval.
- Optional extras: GraphRAG (`graphrag>=3.0.1,<4`), LightRAG via RAG-Anything, Manim for the math animator, Matrix, and numerous Partner IM channel SDKs; several extras are capped below Python 3.14.
- Distribution: PyPI wheel plus prebuilt Docker images on GHCR (`ghcr.io/hkuds/deeptutor`).

## When to reach for it

- Running a self-hosted, private tutoring workspace against your own or local LLM providers (Ollama, LM Studio, llama.cpp, vLLM, Lemonade).
- Studying across documents with switchable RAG engines and a memory system that persists across sessions.
- Deploying tutoring companions into IM channels via Partners, or wiring in a local Claude Code or Codex as a subagent.
- Multi-user deployments with isolated per-user workspaces and admin controls.

## When *not* to reach for it

- Wanting a hosted, zero-setup service rather than a stack you install and operate yourself.
- Environments that cannot run both a Python backend and a Node.js frontend, since the full app needs both.
- Deployments where running model-generated code is unacceptable without review, given the office skills execute generated scripts in a sandbox (host subprocess execution is enabled by default and must be explicitly disabled).
- Lightweight, single-purpose chat needs where a full learning workspace is more than required.

## Maturity signal

DeepTutor is very actively developed, with an extremely rapid release cadence — from a v1.0 beta in April 2026 through v1.5.2 in July 2026, often multiple releases per week — and a last push within the last day. Despite being created at the end of December 2025, it has drawn a large audience quickly (the README notes 20k stars in 111 days). It is Apache-2.0 with a documented agent-native rewrite of roughly 200k lines, so it is best read as young, fast-moving, and feature-additive rather than settled.

## Alternatives

- Open WebUI — use it when you want a general self-hosted LLM chat interface without the tutoring- and learning-specific workflows.
- AnythingLLM — use it when you primarily need a document-QA and RAG desktop app rather than a structured learning system with memory and mastery practice.
- LightRAG (also from HKUDS) — use it when you only need the RAG engine itself, not the surrounding learning workspace.

## Notes

The office skills (docx, pdf, pptx, xlsx) work by having the model write and execute Python scripts through a sandbox; a restricted subprocess sandbox runs on the host by default (`sandbox_allow_subprocess=true`), with a hardened runner sidecar used under docker-compose. Several optional RAG engines pull heavy transitive dependencies (MinerU via RAG-Anything) and carry `python_version < '3.14'` markers to avoid silent downgrades. The single-container Docker setup only needs port 3782 published, because the in-container Next.js middleware forwards API and WebSocket traffic to the backend server-side.

## Tags

python, typescript, nextjs, fastapi, ai-tutor, education, rag, llm, multi-agent-systems, agents, cli
