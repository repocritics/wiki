# lancedb/lancedb

> An embedded vector database built on the Lance columnar format — think "SQLite for vector and multimodal search" that reads directly off object storage.

[GitHub repo](https://github.com/lancedb/lancedb) ·
[Official website](https://lancedb.com/docs) ·
[License: Apache-2.0](https://github.com/lancedb/lancedb/blob/main/LICENSE)

## Overview

LanceDB is an embedded, serverless vector database first released by the LanceDB team in 2023[^1]. Unlike client/server vector stores (Milvus, Weaviate, Qdrant), it runs in-process as a library — you `pip install` or `npm install` it and open a table over a local directory or an object-store URI. There is no daemon to operate. The closest mental model is SQLite or DuckDB, but for vector similarity search, full-text search, and SQL filtering over multimodal data.

Its defining technical bet is the **Lance columnar format**[^2] — a Rust-implemented, Arrow-compatible on-disk format built specifically for ML workloads. Rather than storing vectors in a bespoke index blob, LanceDB stores rows (vectors, scalar metadata, and raw blobs like images) in Lance files and builds ANN indexes alongside them. Because Lance supports random access and zero-copy versioning, LanceDB can serve queries directly from S3/GCS/Azure without loading the whole dataset into RAM — the differentiator that lets it claim petabyte-scale on commodity storage.

The central tension is **embedded convenience vs. write concurrency**. LanceDB is excellent for read-heavy retrieval and single-writer ingestion, and it removes an entire class of operational burden. But it is a library over a storage format, not a transactional database with a coordinator, so concurrent writers, real-time high-throughput upserts, and multi-tenant isolation are weaker than in server-based competitors. The company monetizes this gap through LanceDB Cloud/Enterprise, a managed control plane over the same format[^3].

## Getting Started

```bash
pip install lancedb          # Python
# or: npm install @lancedb/lancedb   (Node/TypeScript)
```

```python
import lancedb
import numpy as np

db = lancedb.connect("./mydb")           # a directory, or "s3://bucket/prefix"

data = [
    {"id": 1, "vector": np.random.rand(128), "text": "a red bicycle"},
    {"id": 2, "vector": np.random.rand(128), "text": "a blue car"},
]
tbl = db.create_table("items", data=data)

# brute-force by default; build an ANN index once the table is large
results = (
    tbl.search(np.random.rand(128))
       .where("id > 0")                  # SQL predicate pushed down
       .limit(5)
       .to_pandas()
)
```

For datasets past ~100k rows, build an index explicitly: `tbl.create_index(num_partitions=256, num_sub_vectors=16)` creates an IVF_PQ index. Without one, `search` does an exact scan, which is correct but linear in table size.

## Architecture / How It Works

LanceDB is a thin, multi-language surface over two Rust crates: **Lance** (the format + I/O + indexing) and **lancedb** (the table/query API). The Python and TypeScript SDKs are bindings; the Rust crate is the native path. This means query semantics are shared across languages, and the heavy lifting never runs in Python or JS.

- **Storage layout.** A table is a directory of versioned Lance fragments plus a manifest. Each write appends a new version rather than mutating in place, so `checkout`/`restore` to a prior version is a metadata operation — no separate snapshot infrastructure. This is the "automatic versioning" and "zero-copy" the README advertises, and it comes directly from the format, not a bolted-on layer.
- **Vector index.** The default ANN index is IVF_PQ (inverted-file coarse quantization + product quantization of residuals). Indexes live next to the data and are memory-mappable, so a query on object storage fetches only the relevant partitions rather than the whole index. HNSW-family variants have been added over time; IVF_PQ remains the workhorse for large, storage-resident datasets.
- **Full-text search.** Native FTS is provided via a Tantivy-based inverted index (Rust's Lucene analog), enabling BM25 keyword search and hybrid vector+keyword ranking without a second system[^4].
- **Filtering and SQL.** Scalar predicates in `.where(...)` are compiled and pushed down through the scan; DataFusion underpins SQL-style execution. You can pre-filter (restrict candidates before ANN) or post-filter, which materially changes recall on selective queries.
- **Object storage first.** The I/O layer treats S3/GCS/Azure and local disk uniformly. Random-access reads over HTTP are what make "search petabytes without a cluster" plausible, at the cost of per-query latency dominated by object-store round trips.

The coupling story: LanceDB's capabilities are almost entirely inherited from the Lance format. Improvements to compression, index types, or storage backends land in Lance and surface automatically. The flip side is that LanceDB is not portable off Lance — your data is in a format whose only mature reader is this project's own stack.

## Production Notes

- **Single-writer model.** Concurrent writers can conflict; the format uses optimistic concurrency with manifest versioning, and losing writers must retry. Designs that assume many independent processes upserting the same table in real time will hit contention. Batch ingestion from one writer is the happy path.
- **Deletes and updates are soft.** Rows are tombstoned and rewritten on compaction, not deleted in place. Without periodic `compact_files()` / optimize, deleted rows and small fragments accumulate, degrading scan and index performance. Compaction and index re-training are operational chores you own on the OSS/local path.
- **Index freshness.** New rows inserted after an index is built are searched by brute force until you re-index. High-churn tables need a re-index cadence; this is easy to forget and shows up as slow tail latencies.
- **Object-store latency.** Reading from S3 adds tens of milliseconds per query versus local NVMe. For latency-sensitive serving, keep hot data on local SSD or front it with the managed offering; do not expect single-digit-ms p99 straight off cold object storage.
- **Recall tuning is manual.** `nprobes` and `refine_factor` at query time, and `num_partitions`/`num_sub_vectors` at build time, trade recall against latency. Defaults are conservative; production recall usually requires measuring against a ground-truth set.
- **Format churn.** Lance is young and evolving. The format is versioned and generally backward-readable, but pinning `lance`/`lancedb` versions and testing upgrades against real data is advisable, especially for long-lived datasets.
- **OSS vs. Cloud gap.** Distributed indexing, managed compaction, autoscaling, and multi-tenant serving are Cloud/Enterprise features. The embedded library is complete for local and single-node use; expect to build the operational scaffolding yourself if you self-host at scale.

## When to Use / When Not

**Use when:**
- You want vector + full-text + SQL retrieval without running and scaling a database server.
- Your data lives in object storage and you want to query it in place, including datasets far larger than RAM.
- You need dataset versioning/reproducibility for ML (time-travel to the exact data a model trained on).
- You're embedding retrieval into an app, notebook, or edge/desktop process where a network hop to a DB is unwanted.
- Ingestion is batch or single-writer and reads dominate.

**Avoid when:**
- You need high-throughput concurrent writes or real-time multi-writer upserts.
- You require strict multi-tenant isolation and RBAC out of the box (self-hosted OSS).
- You need single-digit-millisecond p99 and can't keep data on local SSD.
- You want a mature, widely-second-sourced storage format — Lance is the only real reader.
- Your workload is a small, static, in-memory index where a library like FAISS is simpler.

## Alternatives

- qdrant/qdrant — use instead when you need a dedicated server with strong concurrent-write throughput, payload filtering, and clustering.
- milvus-io/milvus — use instead for large distributed deployments with a full control plane and horizontal scaling.
- weaviate/weaviate — use instead when you want built-in vectorizer modules, GraphQL, and multi-tenancy as first-class server features.
- facebookresearch/faiss — use instead when you just need a fast in-memory ANN index library and will handle storage/persistence yourself.
- chroma-core/chroma — use instead for the simplest local-first RAG prototyping API when multimodal/object-storage scale isn't a concern.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2023-02-28 | Initial public LanceDB repository, built on the Lance format[^1]. |
| — | 2023 | Python SDK, embedded API, IVF_PQ vector index. |
| — | 2023–2024 | TypeScript/Node and Rust SDKs; object-storage (S3/GCS/Azure) backends. |
| — | 2024 | Native full-text search (Tantivy) and hybrid search; scalar indexing. |
| — | 2024–2025 | LanceDB Cloud/Enterprise managed offering; positioning as "multimodal AI lakehouse"[^3]. |

Specific release version numbers are omitted where not verified; the project publishes frequent releases on GitHub and PyPI/npm.

## References

[^1]: LanceDB GitHub repository — created 2023-02-28. https://github.com/lancedb/lancedb
[^2]: Lance columnar format. https://github.com/lancedb/lance
[^3]: LanceDB documentation and product pages. https://lancedb.com/docs
[^4]: LanceDB full-text search documentation (Tantivy-based). https://docs.lancedb.com

## Tags

rust, python, typescript, vector-database, embedded-database, similarity-search, semantic-search, full-text-search, rag, multimodal, columnar-storage, apache-arrow
