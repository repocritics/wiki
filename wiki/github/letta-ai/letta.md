# letta-ai/letta

> Stateful-agent server descended from the MemGPT paper: agents whose memory the LLM edits itself, persisted in a database rather than a prompt string.

[GitHub repo](https://github.com/letta-ai/letta) ·
[Official docs](https://docs.letta.com/) ·
[License: Apache-2.0](https://github.com/letta-ai/letta/blob/main/LICENSE)

## Overview

Letta is the open-source server that grew out of the 2023 MemGPT research project from UC Berkeley[^1], which framed the LLM context window as the "main memory" of an operating system and everything else as swappable "external memory." The core idea: give the model tools to read and rewrite its own memory, so an agent accumulates state across sessions instead of re-deriving it from a fixed system prompt each turn. The project was renamed from MemGPT to Letta in 2024 when the authors incorporated a company around it[^2].

The defining bet is that *agent state belongs in a database, not in your application code*. A Letta agent is a row (plus vector rows) in Postgres — its persona, its editable memory blocks, its message history, its attached tools — reachable through a REST API. You talk to an agent by ID; the server owns the loop, the memory paging, and the persistence. This is the opposite of the "framework as a library you `import`" model that LangGraph or CrewAI use.

Read the README before adopting: this repository is now described by its own maintainers as the **legacy Letta server** behind the V1 API and SDKs. Active development has moved to a separate TypeScript CLI (`letta-code`), and the recommended self-hosting path is now an "App Server" rather than this codebase[^3]. The concepts below still describe what this repo does, but treat it as a mature-but-branching foundation, not the bleeding edge.

## Getting Started

This repo runs as a server; agents live inside it and you drive them over HTTP.

```bash
pip install -U letta
letta server            # starts the API server on http://localhost:8283
# or: docker run -p 8283:8283 letta/letta:latest
```

```python
from letta_client import Letta

client = Letta(base_url="http://localhost:8283")

# An agent is created server-side and persists after this process exits.
agent = client.agents.create(
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-3-small",
    memory_blocks=[
        {"label": "human",   "value": "Name: Bob. Likes terse answers."},
        {"label": "persona", "value": "I am a helpful assistant who remembers."},
    ],
)

resp = client.agents.messages.create(
    agent_id=agent.id,
    messages=[{"role": "user", "content": "Remember that my dog is named Timber."}],
)
# Next session, a fresh client hitting the same agent_id still knows about Timber.
```

## Architecture / How It Works

The mental model is a memory hierarchy that the agent manages with tools, mirroring virtual memory in an OS:

- **Core memory** — small, always-in-context blocks (typically `persona` and `human`, plus custom labels). The agent edits these with built-in tools (`core_memory_append`, `core_memory_replace`). This is the durable "who am I / who are you" state that survives every turn.
- **Recall memory** — the full conversation history, stored in the database and searchable. The recent slice is in-context; older messages are paged out and retrieved on demand.
- **Archival memory** — an open-ended vector store the agent writes to and queries via tool calls (`archival_memory_insert` / `archival_memory_search`). This is long-term knowledge beyond the context window.

When the context window fills, the server summarizes/evicts rather than hard-failing, and the agent can page the evicted content back via search tools. The agentic loop is a heartbeat: the LLM emits tool calls, the server executes them (including memory edits), feeds results back, and continues until the agent yields. Because every step is persisted, an agent is resumable and inspectable at any point.

State lives in **Postgres with pgvector** for production (SQLite is the zero-config default for local use). Tools can be Python functions registered with the server or external services; models are pluggable across OpenAI, Anthropic, and local/OSS backends. The **Agent Development Environment (ADE)** is a separate GUI for inspecting an agent's memory blocks, message trace, and tool calls — useful because a Letta agent's behavior is hard to reason about from code alone when the state is in a database.

## Production Notes

- **You are running a stateful service, not calling a library.** Backups, migrations, and the Postgres/pgvector instance are now your operational problem. Agent state accretes indefinitely; there is no automatic TTL on archival or recall memory, so growth planning is on you.
- **The self-hosting story is in flux.** With active development moved to `letta-code` and the "App Server," pin a specific version and read release notes before upgrading — the API surface this repo exposes (V1) is stable but no longer where new features land[^3].
- **Latency has a floor.** The memory-managed loop can issue several tool round-trips (search, edit, then answer) per user turn, each an LLM call. Interactive UX needs streaming and a fast model; the "OS paging" behavior trades tokens and latency for continuity.
- **Model choice leaks into behavior.** Self-editing memory depends on reliable tool-calling. Weaker or heavily-quantized local models mis-format memory edits, corrupting core blocks; the abstraction assumes a competent function-calling model.
- **Prompt-injection reaches your memory.** Because the agent writes untrusted conversational content into persistent memory that is replayed every turn, injected instructions can persist across sessions. Treat memory blocks as an attack surface, not just storage.

## When to Use / When Not

**Use when:**
- You want agents that genuinely persist and evolve across sessions without hand-rolling a memory/RAG layer.
- You want agents as a managed backend service (create by API, address by ID) rather than objects inside one process.
- You need to inspect and debug agent memory visually (the ADE workflow).

**Avoid when:**
- You want a stateless, single-shot completion or a thin orchestration library you fully control — the server and database are overhead you don't need.
- You need production support on the newest features today; this repo is the legacy branch and the project's center of gravity has moved.
- You can't run and operate Postgres/pgvector, or you need tight latency budgets per turn.

## Alternatives

- mem0ai/mem0 — use instead when you want just a pluggable memory layer to bolt onto your existing agent, not a full agent runtime.
- langchain-ai/langgraph — use instead when you want to own the control-flow graph and supply your own persistence rather than adopt Letta's opinionated memory model.
- crewai/crewAI — use instead when the problem is multi-agent role orchestration rather than long-lived single-agent memory.
- microsoft/autogen — use instead for multi-agent conversation patterns and research-style experimentation.
- run-llama/llama_index — use instead when the core need is data ingestion and retrieval, with agents as a secondary concern.

## History

| Version | Date | Notes |
|---------|------|-------|
| MemGPT paper | 2023-10 | "MemGPT: Towards LLMs as Operating Systems," UC Berkeley[^1]. |
| Repo created | 2023-10-11 | Public MemGPT repository opened. |
| Rename to Letta | 2024 | MemGPT → Letta; company incorporated, server + REST API focus[^2]. |
| V1 API / SDKs | 2024–2025 | Stateful-agent REST API, `letta_client` SDKs, ADE. |
| letta-code pivot | 2026 | Active development moves to a TS CLI; this repo becomes the legacy server[^3]. |

## References

[^1]: Packer et al., "MemGPT: Towards LLMs as Operating Systems," arXiv:2310.08560, 2023. https://arxiv.org/abs/2310.08560
[^2]: Letta documentation and project site (formerly MemGPT). https://docs.letta.com/
[^3]: Letta README, "legacy Letta server" note directing self-hosting to the App Server and active development to the Letta Agent repo. https://github.com/letta-ai/letta

## Tags

python, ai-agents, llm, agent-memory, stateful-agents, memgpt, rest-api, vector-database, pgvector, agent-framework
