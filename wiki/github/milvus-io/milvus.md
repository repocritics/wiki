# milvus-io/milvus

> Cloud-native vector database for approximate-nearest-neighbor search over billions of embeddings, with a disaggregated storage/compute architecture.

[GitHub repo](https://github.com/milvus-io/milvus) ·
[Official website](https://milvus.io) ·
[License: Apache-2.0](https://github.com/milvus-io/milvus/blob/master/LICENSE)

## Overview

Milvus is a purpose-built vector database written in Go (control plane, orchestration) and C++ (the vector search engine, Knowhere)[^1]. It stores learned vector representations of unstructured data — text, images, audio — alongside scalar fields, and answers approximate-nearest-neighbor (ANN) queries with optional metadata filtering. It was open-sourced by Zilliz in 2019 and donated to the LF AI & Data Foundation as a graduated project; Zilliz remains the dominant contributor and sells the managed version (Zilliz Cloud)[^2].

The defining decision in Milvus is **disaggregation**: unlike a single-binary vector store, the distributed deployment splits into an access layer (proxy), a set of coordinators, and independently scalable worker pools (query, data, index nodes), with metadata in etcd, log/segment data in object storage (S3/MinIO), and a write-ahead log delivered through a message queue (historically Pulsar or Kafka). This is what lets Milvus scale reads and writes independently and reach billion-vector collections — and it is also the source of most of its operational weight. The project mitigates this with two lighter deployment modes so you are not forced into the full topology (see below).

Milvus is used to back retrieval-augmented generation (RAG), semantic and image search, deduplication, and recommendation systems. Since 2.4/2.5 it also handles sparse vectors and BM25 full-text search, positioning it for hybrid dense+sparse retrieval rather than dense-only[^3].

## Getting Started

The fastest path is Milvus Lite, an embedded, in-process build that ships inside the Python SDK — no server, data persisted to a local file:

```bash
pip install -U pymilvus
```

```python
from pymilvus import MilvusClient

# Milvus Lite: pass a local filename. Point uri at a server URL for Standalone/Distributed.
client = MilvusClient("milvus_demo.db")

client.create_collection(collection_name="demo", dimension=768)

client.insert(collection_name="demo", data=data)  # data: list of {"id", "vector", ...}

res = client.search(
    collection_name="demo",
    data=query_vectors,          # one or more query vectors
    limit=2,                     # topK
    output_fields=["text"],
)
```

For anything beyond prototyping, run Standalone (single container, all roles in one process) via Docker Compose, or the Helm/Operator install for the Distributed cluster. The same `MilvusClient` code targets all three by changing `uri` and `token`.

## Architecture / How It Works

Milvus separates the system framework from the search engine. The C++ core, **Knowhere**, wraps FAISS, HNSW, DiskANN, and NVIDIA's CAGRA GPU index behind a uniform interface; the Go layer handles the distributed system around it.

The four functional layers:

1. **Access layer (proxy)** — stateless; validates requests, routes them, and merges results. The client entry point.
2. **Coordinators** — root/query/data/index coordinators manage the DDL clock (timestamp oracle), segment allocation, load balancing, and index building. Historically separate processes; newer releases consolidate them.
3. **Worker nodes** — query nodes (serve search over loaded segments), data nodes (consume the write log, flush segments to object storage), index nodes (build indexes asynchronously). Each pool scales independently.
4. **Storage** — etcd for metadata and service discovery; object storage (S3/MinIO/GCS) for log snapshots and sealed segments; a log broker (Pulsar/Kafka, or the newer native WAL) as the streaming backbone.

Writes go to the message queue first (log-as-data), so recently inserted vectors are searchable from **growing segments** in memory before they are flushed and compacted into **sealed segments** that get indexed. This is how Milvus offers real-time freshness while still building heavyweight ANN indexes offline. Search consistency is tunable per query across four levels — Strong, Bounded (staleness window), Session, and Eventually — trading freshness for latency[^4].

## Production Notes

**Pick the deployment mode deliberately.** Milvus Lite is for prototyping and small local datasets only (single process, no concurrency story). Standalone is one container and fine into the low tens of millions of vectors. The Distributed cluster is what the marketing scale numbers refer to, and it brings etcd, object storage, and a message queue as hard dependencies — three stateful systems to run, monitor, and back up in addition to Milvus itself.

**Memory is the real budget.** Most ANN indexes (HNSW, IVF variants) are served from RAM; a collection must be explicitly `load()`ed into query-node memory before it is searchable, and the resident footprint — vectors plus index structure — usually dominates cost. Mitigations: `mmap` to page index/data from disk, DiskANN for SSD-resident indexes, quantization (SQ/PQ/RaBitQ), and hot/cold tiering. Under-provisioning query-node memory produces load failures or OOM, not graceful degradation.

**Too many collections/partitions is the classic footgun.** Each collection and partition carries fixed metadata and segment-management overhead; designs that map one tenant to one collection fall over at thousands of tenants. The intended pattern for high tenant counts is the partition-key field (logical isolation inside one collection), not thousands of physical collections.

**Segment and index lifecycle.** Freshly inserted data lives in small growing segments; compaction merges them and indexes are built asynchronously, so query performance and recall improve minutes after a bulk load rather than instantly. Deletes are soft (tombstones) reclaimed at compaction — high-churn upsert workloads accumulate space until compaction catches up.

**Upgrades and version churn.** The 1.x → 2.x transition was a full rewrite (see History) with no in-place migration path. Within 2.x, minor versions have changed component topology and defaults; read the release notes before upgrading a cluster, and prefer the Operator/Helm managed upgrade over hand-rolled restarts. The Go/C++ split also makes building from source non-trivial (CMake, a C++ toolchain, and a specific Go version).

## When to Use / When Not

**Use when:**
- You need ANN search over hundreds of millions to billions of vectors with horizontal scaling.
- You want dense + sparse + full-text (BM25) hybrid retrieval in one system.
- You need tunable consistency, GPU indexing, or hot/cold storage tiering.
- You are building RAG/search infrastructure and can run (or pay for) a stateful distributed system.

**Avoid when:**
- Your dataset is small (< a few million vectors) — Standalone works, but pgvector or an embedded store is far less to operate.
- You want a single self-contained binary with minimal moving parts — the distributed dependency chain (etcd + object store + message queue) is heavy.
- You already run Postgres or Elasticsearch/OpenSearch and your scale fits their vector support — adding a second stateful system rarely pays off.
- You need embedded/edge-only deployment beyond what Milvus Lite's prototype scope covers.

## Alternatives

- qdrant/qdrant — Rust, single-binary, simpler ops; use when you want vector search without the distributed dependency chain and your scale fits one node (or its lighter cluster).
- weaviate/weaviate — Go vector DB with built-in vectorizer modules and GraphQL; use when you want embedding generation and object storage bundled in.
- pgvector/pgvector — Postgres extension; use when you already run Postgres and want vectors next to relational data without a new system.
- chroma-core/chroma — embedded, developer-first; use for local RAG prototypes and notebooks rather than production scale.
- facebookresearch/faiss — a library, not a database; use when you want to embed ANN search directly and build persistence/serving yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2019-10 | First open-source release; single-node, FAISS-backed. |
| 1.0 | 2021-03 | First stable line; still a monolithic architecture. |
| 2.0 | 2022-01 | Ground-up rewrite: cloud-native, disaggregated storage/compute, log-as-data[^1]. |
| 2.1–2.2 | 2022 | Embedded fields, RBAC, GPU work, dynamic schema. |
| 2.3 | 2023 | GPU indexing (CAGRA), upsert, improved scalar filtering. |
| 2.4 | 2024 | Multi-vector, sparse-vector support, inverted/GPU index advances[^3]. |
| 2.5 | 2024-12 | Native full-text search (BM25), further sparse/hybrid tooling. |

## References

[^1]: Wang et al., "Milvus: A Purpose-Built Vector Data Management System," SIGMOD 2021. https://www.cs.purdue.edu/homes/csjgwang/pubs/SIGMOD21_Milvus.pdf
[^2]: LF AI & Data Foundation — Milvus project page. https://lfaidata.foundation/projects/milvus/
[^3]: Milvus documentation — full-text and hybrid search. https://milvus.io/docs/full-text-search.md
[^4]: Milvus documentation — consistency levels. https://milvus.io/docs/consistency.md

## Tags

vector-database, ann-search, golang, cpp, rag, embeddings, similarity-search, cloud-native, distributed, hnsw, diskann, llm-infrastructure
