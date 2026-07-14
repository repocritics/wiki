# weaviate/weaviate

> An open-source vector database written in Go, with a pluggable module system for turning raw objects into embeddings and searching them.

[GitHub repo](https://github.com/weaviate/weaviate) ·
[Official website](https://weaviate.io/developers/weaviate/) ·
[License: BSD-3-Clause](https://github.com/weaviate/weaviate/blob/main/LICENSE)

## Overview

Weaviate is a vector database: it stores objects together with one or more vector
embeddings and answers approximate-nearest-neighbor (ANN) queries over them, so you
can retrieve records by semantic similarity rather than exact keyword match. It is
aimed at retrieval-augmented generation (RAG), semantic and image search,
recommendation, and classification workloads. The project began around 2016 as a
knowledge-graph experiment and pivoted to vector search, reaching a 1.0 release in
early 2021[^1]; the company behind it (formerly SeMI Technologies) is now Weaviate.

Its defining design choice is the **module system**. Unlike a bare index library,
Weaviate can call an embedding provider (OpenAI, Cohere, HuggingFace, a local
model container, etc.) at import and query time, so the database — not your
application — owns vectorization. This is convenient for prototyping and keeps
query and index vectors consistent, but it couples the database to model-provider
availability, cost, and latency, and it is the main thing that distinguishes
Weaviate from lower-level engines like Qdrant or FAISS.

The second tension is operational. The HNSW index that powers fast search is held
in memory, so RAM — not disk — is usually the scaling limit, and honest capacity
planning means budgeting for quantization from the start.

## Getting Started

```yaml
# docker-compose.yml — Weaviate plus a small local embedding model
services:
  weaviate:
    image: cr.weaviate.io/semitechnologies/weaviate:1.36.0
    ports: ["8080:8080", "50051:50051"]   # REST/GraphQL on 8080, gRPC on 50051
    environment:
      ENABLE_MODULES: text2vec-model2vec
      MODEL2VEC_INFERENCE_API: http://text2vec-model2vec:8080
  text2vec-model2vec:
    image: cr.weaviate.io/semitechnologies/model2vec-inference:minishlab-potion-base-32M
```

```bash
docker compose up -d
pip install -U weaviate-client   # v4 client — gRPC-based, not compatible with v3
```

```python
import weaviate
from weaviate.classes.config import Configure, DataType, Property

client = weaviate.connect_to_local()
client.collections.create(
    name="Article",
    properties=[Property(name="content", data_type=DataType.TEXT)],
    vector_config=Configure.Vectors.text2vec_model2vec(),  # DB vectorizes on import
)
articles = client.collections.get("Article")
articles.data.insert_many([
    {"content": "Vector databases enable semantic search"},
    {"content": "Weaviate supports hybrid search"},
])
print(articles.query.near_text(query="search by meaning", limit=1).objects[0])
client.close()
```

## Architecture / How It Works

Weaviate is a single Go binary that exposes three query surfaces over the same
store: a **REST** API, a **GraphQL** API (the original and most expressive query
language), and a **gRPC** API added later to cut the serialization overhead of
large batch imports and result sets[^2]. The v4 Python and modern TS clients speak
gRPC by default; this is why upgrading from the v3 client is a rewrite, not a bump.

Data is organized into **collections** (historically "classes"), each with a
schema of typed properties and a vector configuration. Objects can carry multiple
named vectors, enabling different embeddings (e.g. text and image) on one object.
The default vector index is **HNSW** — a graph-based ANN index that trades memory
for query speed; a flat/brute-force index exists for small collections, and a
disk-backed option targets larger-than-RAM cases. HNSW graphs live in memory, so
memory footprint scales with vector count × dimensions unless you compress.

**Compression** is first-class and load-bearing: product quantization (PQ), binary
quantization (BQ), and scalar quantization (SQ) shrink the in-memory vectors, often
by an order of magnitude, at some recall cost. Treating quantization as optional is
the most common way to be surprised by RAM usage.

Cluster metadata (the schema and sharding/tenancy state) is replicated with a
**Raft** consensus layer, added to make schema changes consistent across a cluster;
object data replication is separate and offers tunable consistency
(ONE/QUORUM/ALL) rather than the strong consistency of the metadata layer[^3].
**Multi-tenancy** shards a collection into many lightweight tenants, each an
isolated index, which is the intended pattern for per-customer SaaS isolation and
for cold/warm tenant offloading.

Hybrid search fuses BM25 keyword scoring with vector similarity (via reciprocal
rank fusion or relative score fusion) in one call, and generative/rerank modules
can post-process results with an LLM inside the query path.

## Production Notes

- **RAM is the budget.** HNSW is in-memory. Plan capacity as vectors × dimensions ×
  4 bytes plus graph overhead, then divide by your quantization ratio. Enabling PQ/BQ
  after you are already memory-pressured requires a reindex, so decide early.
- **The vectorizer is a runtime dependency.** If the database owns vectorization,
  an outage or rate limit at OpenAI/Cohere becomes an import/query outage. For
  predictable latency and cost, many teams vectorize in their own pipeline and import
  precomputed vectors (`self_provided`), using Weaviate purely as the index.
- **Client-version churn.** The v3→v4 Python client was a hard break (new API,
  gRPC transport). Pin client and server versions together; a v4 client needs the
  gRPC port (50051) reachable, which is a frequent networking/proxy footgun.
- **gRPC port exposure.** Load balancers and ingress that only forward 8080 will
  make the client appear to connect (REST) but fail on real queries. Expose 50051.
- **Recall is a tuning surface, not a given.** HNSW `ef`/`efConstruction` and
  quantization settings trade recall against latency and memory; measure recall on
  your own data. Filtered ANN (a strict metadata filter plus vector search) can
  degrade recall or speed with selectivity, so test real filter cardinalities.
- **Backups and migrations.** Use the backup module (filesystem/S3/GCS) rather than
  copying the data directory; some upgrades trigger index or schema migrations on
  first boot, so stage upgrades and read release notes before jumping versions.

## When to Use / When Not

**Use when:**
- You want the database to handle vectorization, hybrid (BM25 + vector) search, and
  optional reranking/RAG in a single query, without wiring an embedding pipeline.
- You need per-tenant isolation (multi-tenancy) for a SaaS-style product.
- You want an open-source core you can self-host, with a managed cloud as a fallback.

**Avoid when:**
- Your dataset is small or you want an embedded/in-process store — Chroma or FAISS
  are lighter, and you may not need a server at all.
- You already run Postgres and vectors are a secondary concern — pgvector avoids a
  new system to operate.
- You want minimal moving parts and prefer to own embedding yourself with a
  single-purpose, memory-lean engine — Qdrant is a closer fit.
- Your workload is billions of vectors demanding a purpose-built distributed
  architecture — evaluate Milvus alongside Weaviate.

## Alternatives

- qdrant/qdrant — Rust vector engine; use it when you want a lean, filter-strong
  index and prefer to own embedding in your own pipeline.
- milvus-io/milvus — heavier distributed vector DB; use it at very large scale where
  a purpose-built cluster architecture matters more than an integrated module system.
- pgvector/pgvector — a Postgres extension; use it when you already run Postgres and
  vectors are one feature among many, not the whole system.
- chroma-core/chroma — lightweight, embeddable; use it for local prototyping and
  small RAG apps where running a server is overkill.
- elastic/elasticsearch — use it when full-text search is primary and dense-vector
  search is an addition to an existing keyword-search deployment.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-03 | Repository created; project starts as a knowledge-graph effort[^4]. |
| 1.0 | 2021-01 | First stable release after the pivot to vector search[^1]. |
| ~1.20 | 2023 | Multi-tenancy and the gRPC API land, reducing batch/query overhead[^2]. |
| ~1.25 | 2024 | Raft-based cluster metadata for consistent schema changes[^3]. |
| 1.36.0 | 2026 | Version referenced in the current README quickstart. |

## References

[^1]: Weaviate blog / release history — pivot from knowledge graph to vector search and 1.0 GA. https://weaviate.io/blog
[^2]: Weaviate gRPC API documentation. https://docs.weaviate.io/weaviate/api/grpc
[^3]: Weaviate replication and consistency documentation. https://docs.weaviate.io/deploy/configuration/replication
[^4]: GitHub repository metadata, `weaviate/weaviate`, created 2016-03-30. https://github.com/weaviate/weaviate

## Tags

go, vector-database, semantic-search, hnsw, approximate-nearest-neighbor, rag, hybrid-search, embeddings, grpc, cloud-native, information-retrieval
