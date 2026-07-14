# qdrant/qdrant

> A Rust vector database whose defining bet is filtering *inside* the HNSW graph traversal, not before or after it.

[GitHub repo](https://github.com/qdrant/qdrant) ·
[Official website](https://qdrant.tech) ·
[License: Apache-2.0](https://github.com/qdrant/qdrant/blob/master/LICENSE)

## Overview

Qdrant (pronounced "quadrant") is a vector similarity search engine and vector
database written in Rust, first open-sourced in 2021 and reaching a 1.0 release
in February 2023[^1]. It stores *points* — a vector plus an arbitrary JSON
payload — and answers approximate-nearest-neighbor queries over them using an
HNSW index, with rich structured filtering on the payload. It is one of the
handful of purpose-built vector databases (alongside Weaviate and Milvus) that
emerged before the 2023 LLM/RAG wave and then rode it; at ~33k stars it is among
the most-adopted of that cohort.

Its defining technical claim is *filterable HNSW*: rather than pre-filtering
(brute-force scanning matching points) or post-filtering (running ANN then
discarding non-matches, which wrecks recall when the filter is selective),
Qdrant weaves payload conditions into the graph search itself and augments the
HNSW graph with extra links so it stays navigable under filtering[^2]. This is
the feature the project is built around and the reason to pick it over bolting a
vector index onto an existing datastore.

The tension to understand going in: Qdrant is a single-purpose engine, not a
general database. It is developed by a commercial company (Qdrant Solutions
GmbH, Berlin) that also sells Qdrant Cloud, so the OSS core is genuinely
Apache-2.0 and self-hostable, but the managed offering is where operational
sharp edges get smoothed. Note the README's caveat that the *development* branch
is `dev`, not `master` — contributions target `dev`.

## Getting Started

```bash
docker run -p 6333:6333 -p 6334:6334 qdrant/qdrant
```

The bare container above runs with **no authentication, bound to all
interfaces** — fine locally, dangerous on a public host. Set an API key and TLS
before exposing it[^3].

```python
from qdrant_client import QdrantClient, models

client = QdrantClient(url="http://localhost:6333")

client.create_collection(
    collection_name="docs",
    vectors_config=models.VectorParams(size=384, distance=models.Distance.COSINE),
)

client.upsert(
    collection_name="docs",
    points=[
        models.PointStruct(id=1, vector=[0.1] * 384, payload={"lang": "en"}),
    ],
)

hits = client.query_points(
    collection_name="docs",
    query=[0.1] * 384,
    query_filter=models.Filter(
        must=[models.FieldCondition(key="lang", match=models.MatchValue(value="en"))]
    ),
    limit=5,
).points
```

## Architecture / How It Works

A **collection** is the top-level unit and is split into **segments**. Each
segment holds a vector index (HNSW), a payload store, an ID mapper, and its own
deletion flags. Writes land in the newest segment and in a **write-ahead log**;
a background **optimizer** later merges small segments, builds HNSW indexes once
a segment crosses `indexing_threshold`, and applies quantization[^4]. Below that
threshold a segment is searched by brute force — which is why very small
collections feel exact and large ones do not.

Vectors can live in RAM or be memory-mapped from disk (`on_disk: true`);
payloads and the HNSW graph can be independently placed on disk. This
per-component storage control is central to Qdrant's cost story: you trade
latency for RAM.

**Filtering** is the interesting part. To keep HNSW navigable when a query
filter removes most nodes, Qdrant builds a payload index for the filtered field
and adds extra graph edges so the search does not get stranded in a disconnected
region. For very selective filters it can fall back to a plain payload-index
scan. Effective filtering therefore *requires* creating payload indexes on the
fields you filter — an unindexed filter still works but degrades to a slow scan.

**Quantization** comes in three forms: scalar (float32→int8, ~4x smaller),
product quantization (higher compression, slower), and binary quantization (~32x
smaller, well-matched to high-dimension OpenAI-style embeddings). Quantized
vectors are searched fast; a rescoring pass reads the original vectors (kept on
disk) to restore precision[^5].

**Distribution** uses sharding plus replication. Collection-level metadata and
cluster topology are coordinated through a **Raft** consensus layer; the actual
vector data is replicated per shard with a tunable write consistency factor and
read consistency, not through Raft. This split means metadata is strongly
consistent while data replication is eventually consistent under partition.

Interfaces are REST (OpenAPI 3.0) on 6333 and **gRPC** on 6334; production
traffic is expected to use gRPC. Qdrant also supports sparse vectors (for
keyword/BM25-style retrieval), multivectors for late-interaction models like
ColBERT, and hybrid search that fuses results with Reciprocal Rank Fusion or
Distribution-Based Score Fusion.

## Production Notes

- **Unindexed filters are a silent footgun.** Filtering on a payload field
  without a payload index does not error — it quietly scans. Create indexes for
  every field you filter or you will chase phantom latency.
- **Recall is not free under filtering.** Selective filters and low `hnsw_ef`
  interact; tune `hnsw_ef` (search-time) and the `m`/`ef_construct` build
  parameters per collection, and measure recall against a ground-truth set
  rather than trusting defaults.
- **RAM is the real cost.** Unquantized float32 vectors default to memory. A
  large collection can quietly exceed host RAM; plan for on-disk vectors +
  quantization, or size the box accordingly. Binary quantization plus on-disk
  originals is the common cost-cutting recipe.
- **Optimizer spikes.** Segment merges and index builds run in the background
  and consume CPU and disk I/O; large ingest jobs can cause latency jitter while
  optimization catches up. `indexing_threshold` and optimizer config gate this.
- **Snapshots, not point-in-time.** Backup is via full collection/shard
  snapshots. There is no incremental or streaming backup in the OSS engine;
  large collections mean large snapshot copies.
- **Resharding is comparatively recent.** Early versions fixed shard count at
  creation; dynamic resharding of an existing collection arrived later and is
  still an operation to test before relying on it. Verify the behavior on your
  version.
- **Security is opt-in.** No auth by default, API-key auth is a single static
  key (JWT-based read/write scoping exists but is coarse), and TLS is manual
  config. Never expose a default container.

## When to Use / When Not

**Use when:**
- Your queries combine vector similarity with non-trivial structured filters and
  you care about recall under those filters.
- You want a dedicated, self-hostable engine with fine-grained RAM/disk control
  and quantization to manage cost at scale.
- You need sparse + dense + multivector/hybrid retrieval in one system.

**Avoid when:**
- Your data already lives in Postgres and vector search is a side feature —
  pgvector avoids running a second stateful system.
- You need a full document/search database (BM25, aggregations, analytics) as
  the primary workload — Elasticsearch/OpenSearch or Weaviate fit better.
- You want built-in embedding generation and a batteries-included object model;
  Qdrant is deliberately an index, not an application framework.

## Alternatives

- weaviate/weaviate — use instead when you want built-in vectorizer modules and
  a GraphQL/object-oriented data model rather than a bare index.
- milvus-io/milvus — use instead at very large scale with a disaggregated
  storage/compute architecture and you can operate its heavier dependency stack.
- pgvector/pgvector — use instead when your vectors belong next to relational
  data and one Postgres is preferable to a second database.
- chroma-core/chroma — use instead for local, embedded, prototyping-first RAG
  where developer ergonomics beat scale.
- redis/redis — use instead when you already run Redis and want vector search as
  an add-on with in-memory latency.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial OSS | 2021 | First public release of the Rust engine[^1]. |
| 1.0 | 2023-02 | Production-ready GA; API stability commitment[^1]. |
| 1.x | 2023–2024 | Sparse vectors, quantization (scalar/product/binary), multivector/ColBERT, distributed sharding + replication. |
| 1.x | 2024–2026 | Hybrid search fusion (RRF/DBSF), GPU-accelerated indexing, `io_uring` async I/O, resharding, Qdrant Edge (in-process embedded mode)[^6]. |

## References

[^1]: Qdrant blog, "Qdrant 1.0 released" — 2023-02. https://qdrant.tech/blog/qdrant-1.0-release/
[^2]: Qdrant documentation, "Filtering / Filterable HNSW". https://qdrant.tech/documentation/concepts/filtering/
[^3]: Qdrant documentation, "Security — Secure your instance". https://qdrant.tech/documentation/guides/security/
[^4]: Qdrant documentation, "Storage / Segments and Optimizer". https://qdrant.tech/documentation/concepts/storage/
[^5]: Qdrant documentation, "Quantization". https://qdrant.tech/documentation/guides/quantization/
[^6]: Qdrant documentation, "Qdrant Edge". https://qdrant.tech/documentation/edge/

## Tags

rust, vector-database, vector-search, hnsw, approximate-nearest-neighbor, embeddings, semantic-search, rag, similarity-search, hybrid-search, database
