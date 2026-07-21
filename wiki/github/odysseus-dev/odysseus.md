# odysseus-dev/odysseus

A self-hosted AI workspace that bundles chat, agents, research, documents, email, notes, and calendar into a single Docker-deployable app.

## What it is

Odysseus is a self-hosted workspace built around local and API-based language models. It combines chat and agents with research, document editing, email, notes, tasks, and a calendar so that these AI-assisted workflows live in one place you run yourself rather than across separate hosted services. It targets people who want to keep their data and model workflows on their own infrastructure.

## Key features

- Chat and agents with support for local or API models, tools, MCP, file and shell access, skills, and memory.
- A "Cookbook" that gives hardware-aware model recommendations and handles downloads and serving.
- Deep Research for multi-step web research with source reading and report generation, plus a "Compare" mode for blind side-by-side model testing.
- A writing-first document editor with AI edits, suggestions, and Markdown/HTML/CSV support.
- An IMAP/SMTP email inbox with triage, tags, summaries, reminders, and reply drafts.
- Notes, tasks, and a calendar with scheduled agent tasks and CalDAV sync, plus extras like a gallery/image editor, web search, presets, and 2FA.

## Tech stack

- Python, served with FastAPI and uvicorn.
- Pydantic (>=2.13.4) and pydantic-settings (>=2.14.1) for models and configuration, SQLAlchemy for persistence.
- ChromaDB HTTP client plus fastembed for local ONNX embeddings, powering RAG, semantic memory, and tool selection, with a keyword fallback.
- MCP integration, CalDAV/icalendar for calendar sync, nh3 for sanitizing rendered research reports, and markdown rendering.
- Auth and security via cryptography, bcrypt, and pyotp (2FA), with croniter for scheduling.
- Deployed through Docker Compose (with GPU compose variants for AMD and NVIDIA); native macOS/Windows builds exist via a PyInstaller spec and platform build scripts.

## When to reach for it

- You want one self-hosted place for chat, agents, research, documents, email, and calendar rather than several separate tools.
- You need local model workflows and hardware-aware guidance on which models to run.
- You want AI-assisted email triage, document editing, and scheduled agent tasks under your own control.

## When *not* to reach for it

- You only need a lightweight single-purpose chat UI; the workspace scope is broad.
- You do not want to run and secure your own deployment (Docker, auth, ports, backups).
- AGPL-3.0 does not fit your distribution or integration plans.
- You require a managed SaaS with support guarantees rather than a self-hosted app.

## Maturity signal

The project has drawn a very large star count in a short window (created in mid-2026, with the snapshot taken the same day as its most recent push), which points to fast, active development and rapid adoption rather than a settled codebase. The default branch is `dev`, and the open-issue count is high relative to the project's age, both consistent with a fast-moving early-stage project still stabilizing. AGPL-3.0 and the not-archived status indicate an actively maintained, copyleft-licensed effort.

## Alternatives

- Open WebUI — use it instead when you mainly want a self-hosted chat interface for local and API models without the surrounding email/calendar/document suite.
- LibreChat — use it instead when you want a self-hosted multi-provider chat app and don't need the broader workspace features.
- AnythingLLM — use it instead when your focus is a self-hosted document/RAG workspace with agents rather than an all-in-one productivity suite.

## Notes

Vector store and local embeddings are treated as core dependencies rather than optional add-ons because they sit on the main agent paths, though the app degrades to keyword fallback if they are missing. The repository ships integrations for Claude and Codex as skills/plugins, and keeps a documented threat model and security CI alongside the app.

## Tags

self-hosted, ai-workspace, llm, agents, rag, mcp, fastapi, python, docker, chat
