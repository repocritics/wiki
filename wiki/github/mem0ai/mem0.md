# mem0ai/mem0

> A memory layer for AI agents: an LLM extracts durable facts from conversations and stores them in a vector (optionally graph) database for later retrieval.

[GitHub repo](https://github.com/mem0ai/mem0) ·
[Official website](https://mem0.ai) ·
[License: Apache-2.0](https://github.com/mem0ai/mem0/blob/main/LICENSE)

## Overview

Mem0 ("mem-zero") is a memory layer that sits between an application and its LLM. Instead of replaying an entire chat history on every turn, an application writes conversation turns to Mem0, which uses an LLM to distill them into short, standalone facts ("prefers dark mode", "is allergic to penicillin"), stores those facts in a vector store, and returns the most relevant ones at query time. The pitch is token economy and continuity: retrieve a handful of pertinent memories rather than stuffing a growing transcript into the context window[^1].

The project is built by the team behind Embedchain, a RAG framework whose repository (created mid-2023) was reoriented into Mem0; the company went through Y Combinator's S24 batch[^2]. As of 2026 it is one of the most-starred projects in the "agent memory" category, with roughly 60k stars and very active development (commits daily). It is also a commercial product: an open-source SDK sits alongside a hosted "Mem0 Platform," and the two are not feature-equivalent.

That split is the defining tension. The headline benchmark numbers in the README (LoCoMo, LongMemEval, BEAM) are explicitly measured on the managed platform, "which includes proprietary optimizations not available in the open-source SDK"[^3]. The OSS library is genuinely usable and self-hostable, but the marketing gravity pulls toward the paid tier, and some retrieval quality shown in the paper and benchmarks does not ship in the box. Read Mem0 as an SDK with a good default pipeline plus an upsell, not as a batteries-included clone of the hosted service.

## Getting Started

```bash
pip install mem0ai
# optional hybrid search (BM25 + entity extraction):
pip install "mem0ai[nlp]" && python -m spacy download en_core_web_sm
```

```python
from mem0 import Memory

memory = Memory()  # defaults to OpenAI LLM + embeddings; set OPENAI_API_KEY

# Write: the LLM extracts durable facts from these turns
memory.add(
    [
        {"role": "user", "content": "I'm vegetarian and I live in Seoul"},
        {"role": "assistant", "content": "Noted!"},
    ],
    user_id="alice",
)

# Read: semantic retrieval of relevant memories
hits = memory.search(query="what should I cook for Alice?", user_id="alice", top_k=3)
for m in hits["results"]:
    print(m["memory"])   # -> "Is vegetarian", "Lives in Seoul"
```

A TypeScript SDK (`npm install mem0ai`) and a self-hosted server (`cd server && docker compose up`) cover the same API surface.

## Architecture / How It Works

Mem0's core is an **extract-then-reconcile write pipeline**, not a plain vector insert:

1. **Extract** — on `add()`, an LLM reads the new turns and emits a list of candidate facts as short declarative statements.
2. **Reconcile** — historically, a second LLM pass compared each candidate against semantically similar existing memories and chose an operation: ADD (new), UPDATE (supersede a stale fact), DELETE (contradiction), or NOOP. This is what lets "moved to Seoul" overwrite "lives in Busan" rather than accumulating both.
3. **Store** — facts are embedded and written to a vector store. Metadata (`user_id`, `agent_id`, `run_id`) scopes retrieval to a user, an agent, or a session.
4. **Retrieve** — `search()` embeds the query and returns top-k nearest facts, filtered by scope.

The **v3 algorithm (April 2026)** changes step 2 to a single-pass, ADD-only extraction: one LLM call, no UPDATE/DELETE, memories accumulate rather than being overwritten, with retrieval doing more of the work via fused semantic + BM25 keyword + entity-match scoring and temporal ranking[^3]. This trades write-time reconciliation cost for read-time ranking complexity, and it means a fresh install behaves differently from older tutorials.

**Storage is pluggable.** The vector store, LLM, and embedder are all swappable via config. Qdrant is the default vector backend; Chroma, pgvector, Pinecone, Weaviate, Milvus, and others are supported. The LLM and embedder default to OpenAI but can be pointed at Anthropic, local models via Ollama, and so on[^4].

**Graph memory** is an optional mode: alongside (or instead of) plain vector facts, Mem0 extracts entities and relationships and writes them to a graph database (Neo4j, Memgraph). This improves multi-hop questions ("who introduced Alice to Bob?") at the cost of a second store to operate and more LLM calls per write.

The unavoidable architectural fact: **every `add()` costs at least one LLM call.** Mem0 is not a passive store; it is an LLM pipeline with a database attached. That is the source of both its quality and its cost.

## Production Notes

- **Write latency and cost are real.** Because extraction (and, pre-v3, reconciliation) invokes an LLM, `add()` is slow and metered — not a millisecond database write. High-write workloads (logging every turn) can dominate both your latency budget and your token bill. Batch turns, or write memories asynchronously off the request path.
- **Extraction quality is model-dependent.** With a weak or heavily quantized local LLM, the fact extractor produces noise, drops obvious facts, or hallucinates. The published benchmarks assume a strong model; do not expect them with a 7B local model.
- **OSS ≠ Platform.** The reproducible, in-box behavior is the SDK's default pipeline. Benchmark scores, some retrieval optimizations, and the dashboard are platform features. Budget for either self-hosting the plumbing yourself or paying for the hosted tier — the open-source path is more DIY than the front page implies[^3].
- **Non-determinism.** The same conversation can yield different stored facts across runs (LLM sampling) and different `UPDATE`/`DELETE` decisions in older versions. Memory contents are not a pure function of input. This complicates testing and audits.
- **Schema/version churn.** Mem0 iterates quickly; the config surface, default models, and the memory algorithm itself (v2 → v3) have changed in ways that alter behavior. Pin the version, read the migration guide before upgrading, and re-validate retrieval quality after a bump[^5].
- **Self-hosted auth changed.** Recent self-hosted server builds ship with auth on by default; upgrading from a pre-auth build requires setting `ADMIN_API_KEY` or explicitly `AUTH_DISABLED=true` for local dev.
- **Language stats mislead.** GitHub reports the monorepo as majority TypeScript (server, JS SDK, docs site), but the reference SDK most tutorials use is Python (`mem0ai` on PyPI). Check which SDK a given example targets.

## When to Use / When Not

**Use when:**
- You have a conversational assistant that should remember user preferences and facts across sessions, and you want that as a library call rather than a bespoke pipeline.
- You want a pluggable memory abstraction over an existing vector store and LLM you already run.
- You want the option of a managed service later without rewriting your integration.

**Avoid when:**
- You need deterministic, auditable memory (regulated domains where "the LLM decided to delete a fact" is unacceptable). A plain database with explicit CRUD is more honest.
- Your "memory" is really document RAG over a static corpus — use a retrieval framework directly; the extraction pipeline adds cost without benefit.
- Write volume is high and latency-sensitive, and you cannot move memory writes off the request path.
- You want the benchmark-quality retrieval out of the box without paying for the platform — measure the OSS defaults on your own data first.

## Alternatives

- letta-ai/letta — the MemGPT lineage; a full stateful-agent runtime with self-editing memory, heavier than a memory library. Use when you want the agent loop and memory managed together, not just a store.
- getzep/zep — memory built on a temporal knowledge graph (Graphiti). Use when relationships and "what was true when" matter more than flat fact recall.
- topoteretes/cognee — memory as ECL (extract-cognify-load) pipelines producing a knowledge graph. Use when you want graph-first semantic memory and control over the pipeline.
- run-llama/llama_index — general retrieval/RAG framework. Use when your problem is search over documents, not distilling conversational facts.
- langchain-ai/langchain — has memory abstractions inside a broader agent toolkit. Use when you are already committed to LangChain and want in-ecosystem memory.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2023-06 | Repository created; Embedchain-era RAG toolkit[^2]. |
| — | 2024 | Reoriented as Mem0; Y Combinator S24 batch[^2]. |
| paper | 2025-04 | "Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory", arXiv:2504.19413[^6]. |
| v3 algorithm | 2026-04 | Single-pass ADD-only extraction; fused semantic/BM25/entity retrieval; temporal ranking[^3]. |

*(Mem0 does not publish a simple linear version-number history; the table records the milestones that changed behavior rather than every SDK release.)*

## References

[^1]: Mem0 README, "Introduction". https://github.com/mem0ai/mem0
[^2]: Mem0 on Y Combinator (S24). https://www.ycombinator.com/companies/mem0
[^3]: Mem0 README, "New Memory Algorithm (April 2026)" — note that headline benchmarks reflect the managed platform, not the OSS SDK. https://github.com/mem0ai/mem0
[^4]: Mem0 docs, supported LLMs / embedders / vector stores. https://docs.mem0.ai
[^5]: Mem0 migration guide (OSS v2 → v3). https://docs.mem0.ai/migration/oss-v2-to-v3
[^6]: Chhikara et al., "Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory", arXiv:2504.19413, 2025. https://arxiv.org/abs/2504.19413

## Tags

python, typescript, ai-agents, long-term-memory, memory-layer, rag, llm, vector-database, knowledge-graph, personalization, agent-infrastructure
