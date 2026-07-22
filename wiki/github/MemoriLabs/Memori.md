# MemoriLabs/Memori

Agent-native memory infrastructure: an LLM-, datastore-, and framework-agnostic layer that turns agent execution and conversation into structured, persistent state.

## What it is

Memori is a memory layer for LLM applications and agents that captures what agents do and say and stores it as structured, recallable state. It is aimed at teams building production agents that need persistent memory without re-architecting their stack — it integrates with existing datastores and frameworks rather than replacing them. You register your LLM client, attribute interactions to an entity and a process, and Memori persists and recalls context automatically in the background. It is offered as a managed cloud and can also run against your own database (BYODB) across managed, single-tenant, VPC, or on-premises deployments.

## Key features

- Drop-in registration around existing LLM clients (Python and TypeScript SDKs) that persists and recalls conversation automatically once you set attribution.
- An attribution model with three levels — entity (a user), process (an agent/program), and session — used to scope and group memories.
- Advanced Augmentation that enriches memories in the background (attributes, events, facts, people, preferences, relationships, rules, skills) with no added request latency.
- Multiple integration surfaces: SDKs, an OpenClaw plugin, a Hermes memory provider with explicit recall tools, and a hosted MCP endpoint for clients such as Claude Code, Cursor, Codex, and Warp.
- Bring-your-own-database support across SQLite, PostgreSQL, MySQL, MongoDB, Oracle, CockroachDB, and TiDB.
- A reported LoCoMo benchmark result of 81.95% accuracy using ~1,294 tokens per query (about 4.97% of a full-context footprint).

## Tech stack

- Python SDK (version 3.3.7, requires Python >= 3.10) and a TypeScript/Node SDK (`@memorilabs/memori`).
- A Rust core (`core/`, built with setuptools-rust) providing storage, retrieval, search, embeddings, and augmentation, with Node and Python bindings; native fastembed embeddings ship in the default wheel.
- Python dependencies include aiohttp, botocore, faiss-cpu, grpcio, numpy, protobuf, and requests, with optional extras for SQLAlchemy, TiDB, and CockroachDB.
- Storage drivers for SQLite, PostgreSQL, and MySQL, plus documented MongoDB, Oracle, CockroachDB, and TiDB via BYODB.
- Supported LLMs: Anthropic, Bedrock, DeepSeek, Gemini, Grok (xAI), and OpenAI; supported frameworks: Agno, LangChain, and Pydantic AI.
- Distributed on PyPI and npm; Apache-2.0 per the README and Python package metadata (GitHub reports the license as unrecognized).

## When to reach for it

- Adding persistent, cross-session memory to an agent or chatbot without changing agent code or prompts.
- Giving coding agents (Claude Code, Cursor, Codex, Warp) shared, accumulating context via the MCP endpoint.
- Keeping memory in a database you already operate (BYODB) rather than adopting a new datastore.
- Enterprise deployments needing single-tenant, VPC, or on-premises hosting of the memory layer.
- Reducing prompt size versus full-context prompting while preserving recall quality.

## When *not* to reach for it

- The zero-config quickstart routes through Memori Cloud and requires a `MEMORI_API_KEY`; if you want a fully self-contained local store with no hosted service, BYODB is the path but adds setup.
- It is memory infrastructure, not an agent framework or a vector database on its own — the wrong layer if you just need raw embedding storage.
- Advanced Augmentation is rate-limited without an account; heavy unauthenticated use will hit limits.
- The GitHub license is reported as NOASSERTION even though the project states Apache-2.0; teams with strict license scanning may need to confirm terms before adopting.

## Maturity signal

Created in July 2025 and pushed in mid-2026, Memori has been developed steadily for roughly a year and has a large, growing audience with a notably high fork count. The Python package is at 3.3.7 yet still classified "Development Status :: 3 - Alpha," a mismatch suggesting the API is iterated on frequently even as version numbers advance. The breadth of integrations (SDKs in two languages, a Rust core, OpenClaw/Hermes/MCP surfaces, and many database drivers) and a published benchmark paper point to an actively maintained, commercially backed project rather than a hobby effort.

## Alternatives

- Mem0 — use it when you want an established open-source memory layer; the README positions Memori as outperforming it on LoCoMo, so compare directly on your workload.
- Zep — use it when you want a memory service with its own retrieval stack; Memori claims smaller prompts versus Zep on the benchmark.
- LangMem — use it when you are already in the LangChain ecosystem and want its native memory tooling.

## Notes

- Memori emphasizes memory "from what agents do, not just what they say" — it captures tool calls, decisions, and outcomes from agent execution, not only chat turns.
- Despite being a Python-first SDK, the heavy lifting (storage, retrieval, search, embeddings, augmentation) is implemented in a Rust core with language bindings, and embeddings run natively rather than via a separate service.
- Without attribution (entity and process), Memori will not create memories at all.

## Tags

python, typescript, rust, llm, agent-memory, memory-management, rag, state-management, database, sdk
