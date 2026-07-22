# getzep/graphiti

A Python framework for building and querying temporal context graphs that track how facts change over time for AI agents.

## What it is

Graphiti builds context graphs from structured and unstructured data, where every fact carries a validity window and traces back to the raw episode that produced it. Unlike static knowledge graphs or batch-oriented GraphRAG approaches, it integrates new data incrementally without recomputing the whole graph, and supports historical queries about what was true at a given point in time. It is aimed at developers building context-aware, memory-equipped AI agents that operate on evolving real-world data.

## Key features

- Bi-temporal fact management: facts have validity windows and are invalidated rather than deleted when information changes, so you can query current or historical state.
- Episodes and provenance, with every entity and relationship tracing back to the raw data that produced it.
- Prescribed or learned ontology, with developer-defined entity and edge types via Pydantic models.
- Incremental graph construction that integrates new data immediately without batch recomputation.
- Hybrid retrieval combining semantic embeddings, keyword (BM25), and graph traversal.
- Pluggable graph backends and an included MCP server plus a FastAPI REST service.

## Tech stack

- Primary language: Python, requiring `>=3.10,<4`; distributed as `graphiti-core` (version 0.29.2 in the manifest), built with hatchling.
- Core dependencies include `pydantic>=2.11.5`, `neo4j>=5.26.0`, `openai>=1.91.0`, `tenacity>=9.0.0`, `numpy`, `python-dotenv`, and `posthog`.
- Graph backends via extras: Neo4j (default), FalkorDB (`falkordb>=1.1.2,<2.0.0`), embedded FalkorDB Lite (Python 3.12+), Amazon Neptune, and Kuzu (deprecated).
- LLM and embedding providers via extras: OpenAI by default, with Anthropic, Groq, Google Gemini, Voyage, and OpenAI-compatible or local endpoints supported.

## When to reach for it

- Giving agents structured, evolving memory instead of flat document chunks or raw chat history.
- Applications that need precise historical queries and automatic handling of changed or contradictory facts.
- Continuously ingesting user interactions and enterprise data into a queryable graph in near real time.
- Combining semantic, keyword, and graph-traversal retrieval in one low-latency query path.

## When *not* to reach for it

- One-shot static document summarization, where a batch GraphRAG approach may be simpler.
- Teams unwilling to run and operate a graph database backend and the surrounding retrieval system themselves.
- Pipelines relying on smaller or local models that do not reliably honor structured output, which the README warns can cause ingestion failures.
- Cases where you want a fully managed, turnkey platform rather than a self-hosted OSS core.

## Maturity signal

Graphiti is actively maintained, with a recent last push, an Apache-2.0 license, and a published arXiv paper backing its architecture. It is the open-source engine underneath Zep's commercial context infrastructure, which signals sustained corporate investment. The large open-issue count relative to a project of this size reflects an active, fast-moving codebase and a broad support surface across multiple graph and LLM backends rather than neglect.

## Alternatives

- Microsoft GraphRAG — use it for static document summarization and batch-oriented entity clustering, per the README's own comparison.
- Zep — use the managed platform when you want turnkey, production-ready context retrieval with governance, SLAs, and built-in user and conversation management.
- Mem0 — use it when you want a lighter agent-memory layer without operating a full temporal graph engine.

## Notes

Kuzu support is deprecated because the upstream project is unmaintained, and the driver emits a `DeprecationWarning`; new projects should use Neo4j or FalkorDB. Ingestion concurrency defaults low (`SEMAPHORE_LIMIT=10`) to avoid provider 429 rate-limit errors and can be raised if your provider allows. Graphiti collects anonymous, opt-out telemetry via PostHog, and the README documents exactly what is and is not collected.

## Tags

python, knowledge-graph, temporal-graph, graph-database, llm, rag, agents, agent-memory, neo4j, machine-learning, library, mcp
