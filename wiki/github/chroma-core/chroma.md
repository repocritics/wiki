# chroma-core/chroma

> An open-source embedding database for AI applications — store, index, and query vectors with a four-function API.

[GitHub repo](https://github.com/chroma-core/chroma) ·
[Official website](https://www.trychroma.com/) ·
[License: Apache-2.0](https://github.com/chroma-core/chroma/blob/main/LICENSE)

## Overview

Chroma is a vector database aimed at retrieval-augmented generation (RAG) and agent workloads. Its pitch is developer ergonomics: you hand it text, it tokenizes, embeds, indexes, and lets you query by semantic similarity through an API that is deliberately only four core functions (`create_collection`, `add`, `query`, `get`)[^1]. First released in late 2022, it rode the RAG wave of 2023 and became the default "just works" local vector store for LangChain and LlamaIndex tutorials — which is a large part of how it accumulated adoption.

The defining tension is **prototype ergonomics versus production scale**. Chroma's happy path is `chromadb.Client()` in-memory or a single-file persistent store on a laptop; that same simplicity is why teams outgrow it. Historically the OSS product was a single-node Python server backed by SQLite for metadata and an in-memory HNSW index for vectors — excellent for getting started, awkward once a collection no longer fits comfortably in one machine's RAM. Chroma has since rewritten the single-node core in Rust and offers a separate distributed architecture through its hosted Chroma Cloud, but the split between "the easy embedded thing" and "the scalable thing" is the central fact to understand before adopting it.

The repository's primary language is now Rust[^2], reflecting the rewrite of the core engine; the client surface most users touch remains the Python (`chromadb`) and JavaScript (`chromadb`) packages.

## Getting Started

```bash
pip install chromadb            # Python client + embedded server
# npm install chromadb          # JavaScript/TypeScript client
# chroma run --path /chroma_db  # standalone client-server mode
```

```python
import chromadb

# In-memory for prototyping; use PersistentClient(path=...) to persist to disk.
client = chromadb.Client()
collection = client.create_collection("all-my-documents")

collection.add(
    documents=["This is document1", "This is document2"],  # embedded automatically
    metadatas=[{"source": "notion"}, {"source": "google-docs"}],
    ids=["doc1", "doc2"],
)

results = collection.query(
    query_texts=["This is a query document"],
    n_results=2,
    where={"source": "notion"},          # optional metadata filter
    where_document={"$contains": "doc"},  # optional full-text filter
)
```

If you pass no embeddings, Chroma computes them with a default model (Sentence Transformers `all-MiniLM-L6-v2` run through ONNX Runtime), downloaded on first use. You can supply your own vectors or plug in an embedding function (OpenAI, Cohere, Hugging Face, etc.) instead.

## Architecture / How It Works

A Chroma deployment has a few distinct layers:

1. **Client** — thin Python/JS libraries. The full `chromadb` package also bundles the embedded server; `chromadb-client` is a lighter HTTP-only client for talking to a remote server without the server dependencies.
2. **Metadata store** — SQLite in single-node mode. Holds collections, documents, metadata, and drives metadata filtering (`where`) and full-text filtering (`where_document`, backed by SQLite FTS).
3. **Vector index** — HNSW (Hierarchical Navigable Small World) for approximate nearest-neighbor search, historically via `hnswlib`. The index for a collection is loaded into memory to serve queries.
4. **Segment/execution layer** — the part rewritten in Rust. Query planning and index execution moved off the Python hot path to reduce latency and memory overhead.

The storage layer has changed substantially across the project's life. Early Chroma used DuckDB + Parquet for local persistence and ClickHouse for the server; the 0.4 line replaced that with a segment-based architecture on SQLite[^3]. This matters when reading old tutorials — persistence paths, migration steps, and config from the DuckDB era do not apply to current versions.

Chroma Cloud runs a genuinely different, distributed architecture (object-storage-backed, serverless query nodes) rather than a scaled-up single node. The OSS single-node server and the distributed system share an API but not an implementation, so single-node operational intuition does not transfer directly to the hosted product.

## Production Notes

**HNSW lives in RAM.** The vector index for a queried collection is held in memory. Memory scales with vector count × dimensionality; large collections (tens of millions of high-dimensional vectors) can exhaust a single node. Budget RAM accordingly and treat single-node Chroma as bounded by one machine.

**Single-node has no built-in sharding or replication.** The OSS server is one process against one SQLite metadata store and local index files. There is no first-class horizontal scaling, failover, or multi-writer story in that mode — horizontal scale is the job of the distributed/Cloud deployment. Do not assume `chroma run` gives you a highly available cluster.

**Pre-1.0 API and storage churn.** Chroma iterated fast and shipped breaking changes across minor versions (0.3 → 0.4 → 0.5 → 0.6 each carried migrations or removals). Persistent data written by one minor version sometimes required a migration to be readable by the next. Pin versions, read release notes before upgrading, and back up the persist directory.

**The `chromadb` install is heavy.** It pulls ONNX Runtime and other native dependencies to support default embeddings. In constrained or serverless environments this bloats image size and cold-start time; use `chromadb-client` when the process only needs to talk to a remote server, and disable the default embedding function if you supply your own vectors.

**Telemetry is on by default.** Chroma collects anonymized usage telemetry unless disabled (`ANONYMIZED_TELEMETRY=False` or the client setting). Turn it off explicitly in regulated or air-gapped deployments.

**Distance metric is fixed at collection creation.** The similarity space (`l2`, `cosine`, `ip`) is set via HNSW metadata when the collection is created and is not trivially changed afterward — pick it up front to match how your embeddings were trained.

## When to Use / When Not

**Use when:**
- You are prototyping RAG or agent memory and want a vector store running in one line.
- Your corpus fits comfortably on a single machine and you value ergonomics over operational control.
- You are already in the LangChain / LlamaIndex ecosystem and want the path of least resistance.
- You want to start embedded and later lift-and-shift to a hosted service with the same API (Chroma Cloud).

**Avoid when:**
- You need billions of vectors, sharding, replication, or strict HA from the open-source deployment alone.
- You are already running Postgres and would rather not operate a second datastore — pgvector keeps vectors next to your relational data.
- You need advanced production features (quantization, tiered storage, fine-grained payload indexing) that dedicated engines expose today.
- You require long-term storage-format stability; Chroma's pre-1.0 history involved repeated migrations.

## Alternatives

- qdrant/qdrant — Rust vector DB with quantization, payload filtering, and clustering; use when you need production-scale search with more operational knobs.
- weaviate/weaviate — Go vector DB with hybrid search and a module ecosystem; use when you want built-in hybrid retrieval and a GraphQL surface.
- milvus-io/milvus — distributed vector database built for very large scale; use when you have billions of vectors and a cluster to run them.
- pgvector/pgvector — Postgres extension; use when you already run Postgres and want vectors beside relational data with no new service.
- lancedb/lancedb — embedded, columnar (Lance/Arrow) vector store; use when you want an on-disk embedded engine optimized for larger-than-RAM datasets.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2022-10 | First public release; DuckDB + Parquet local, ClickHouse server[^1]. |
| 0.4 | 2023-07 | Segment-based architecture, SQLite metadata store, revamped persistence[^3]. |
| 0.5 | 2024 | Continued API refinement; performance and reliability work. |
| 0.6 | 2024–2025 | Further breaking cleanups ahead of 1.0. |
| 1.0 | 2025-04 | Single-node core rewritten in Rust[^4]. |

## References

[^1]: Chroma README and core API. https://github.com/chroma-core/chroma
[^2]: GitHub API repository metadata (primary language: Rust; Apache-2.0), retrieved 2026-07-15. https://api.github.com/repos/chroma-core/chroma
[^3]: Chroma documentation — migration and storage/architecture notes. https://docs.trychroma.com/
[^4]: Chroma blog, "Chroma 1.0.0" (Rust rewrite of the single-node core). https://www.trychroma.com/

## Tags

vector-database, embeddings, rag, semantic-search, ai-infrastructure, rust, python, hnsw, retrieval, similarity-search
