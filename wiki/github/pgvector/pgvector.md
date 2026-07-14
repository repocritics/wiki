# pgvector/pgvector

> Vector similarity search as a Postgres extension — embeddings live in the same table as the rest of your data.

[GitHub repo](https://github.com/pgvector/pgvector) ·
[License: PostgreSQL](https://github.com/pgvector/pgvector/blob/master/LICENSE)

## Overview

pgvector is a C extension that adds a `vector` column type and approximate
nearest-neighbor indexing to PostgreSQL. It was written by Andrew Kane and first
released in 2021[^1]. The pitch is not "fastest vector search" — it is "you
already run Postgres, so keep your embeddings next to your rows and query both
with one SQL statement." That framing is the whole product: you get ACID
transactions, JOINs, foreign keys, point-in-time recovery, replication, and the
rest of Postgres for free, and vector search becomes one more index type rather
than a separate service to operate and keep in sync.

The defining tradeoff follows directly. pgvector inherits Postgres's operational
model — single-primary writes, no built-in horizontal sharding, index structures
that must fit alongside everything else in shared memory. Dedicated engines
(Qdrant, Milvus, Weaviate) are built for billion-scale workloads with aggressive
quantization and distributed storage; pgvector targets the far more common case
of a few million to a few tens of millions of embeddings that belong to
relational data you already have. Choosing it is a bet that your vector problem
is smaller than your data-integrity and operational-simplicity problem.

As of 0.8.x it supports exact and approximate search, HNSW and IVFFlat indexes,
full/half/binary/sparse vector types, six distance functions, and iterative
index scans for filtered queries[^2]. It requires Postgres 13+ and ships
preinstalled on most managed Postgres offerings (RDS/Aurora, Cloud SQL, Azure,
Supabase, Neon).

## Getting Started

Build from source (or use Docker, Homebrew, APT/Yum, PGXN, conda-forge):

```sh
cd /tmp
git clone --branch v0.8.5 https://github.com/pgvector/pgvector.git
cd pgvector
make && make install   # may need sudo
```

```sql
CREATE EXTENSION vector;

CREATE TABLE items (id bigserial PRIMARY KEY, embedding vector(3));
INSERT INTO items (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');

-- exact nearest neighbors by L2 distance
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;

-- add an approximate index for scale (trades recall for speed)
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops);
```

Operators: `<->` L2, `<#>` (negative) inner product, `<=>` cosine, `<+>` L1,
`<~>` Hamming, `<%>` Jaccard. Note `<#>` returns the *negative* inner product
because Postgres only supports ascending-order index scans.

## Architecture / How It Works

pgvector registers new types and access methods through Postgres's standard
extension API, so its indexes are real Postgres indexes: they are WAL-logged,
crash-safe, replicated to standbys, and visible to the query planner. There are
two index types with opposite tradeoffs:

- **HNSW** — a multi-layer proximity graph. Better speed/recall tradeoff, can be
  built on an empty table (no training step), but slow to build and memory-hungry
  because the graph plus the vectors it points at must live in the index.
- **IVFFlat** — partitions vectors into `lists` via k-means, then probes the
  nearest lists at query time. Faster to build and smaller, but recall depends on
  having representative data *before* you build (rebuild after major data
  changes) and on choosing `lists` (≈`rows/1000` up to 1M rows, `sqrt(rows)`
  beyond) and `probes` sensibly.

Recall is tuned at query time, not build time: `hnsw.ef_search` (default 40) and
`ivfflat.probes` (default 1) trade latency for accuracy and can be set per
transaction with `SET LOCAL`. Both indexes need one operator-class per distance
function, so an index built for `vector_l2_ops` will not serve cosine queries.

The vector types are the other half. `vector` stores float32 (up to 16,000 dims,
indexable to 2,000), `halfvec` float16 (indexable to 4,000, half the memory),
`bit` binary vectors (to 64,000 dims), and `sparsevec` sparse vectors.
Quantization is done through Postgres expression indexes rather than a bespoke
codec — e.g. `binary_quantize(embedding)::bit(N)` indexed with `bit_hamming_ops`,
then re-ranked against the originals — reusing existing SQL machinery.

## Production Notes

The recurring operator surprises come from the collision of approximate indexes
with Postgres's relational features:

- **Filtered queries silently lose recall.** With an approximate index, a `WHERE`
  clause is applied *after* the graph is scanned. With HNSW's default
  `ef_search = 40`, a filter matching 10% of rows returns ~4 results on average,
  not your `LIMIT`. Enable `hnsw.iterative_scan = strict_order` (or
  `relaxed_order` for better recall) — added in 0.8.0 — to keep scanning until
  enough rows match, bounded by `hnsw.max_scan_tuples`. Before 0.8.0 the standard
  workarounds were partial indexes or partitioning by the filter column.
- **Index builds are the main cost.** HNSW build time explodes once the graph no
  longer fits in `maintenance_work_mem` (pgvector emits a `NOTICE` when this
  happens). Raise `maintenance_work_mem`, bump `max_parallel_maintenance_workers`
  (default 2), and always build indexes *after* bulk-loading via `COPY`. On large
  datasets, building HNSW can take hours and pin CPU.
- **Memory footprint is real.** HNSW keeps full vectors in the index; a few
  million 1536-dim float32 vectors is multiple GB competing for `shared_buffers`.
  `halfvec` indexing and binary quantization exist precisely to shrink this.
- **No native distributed scaling.** Sharding means Citus or application-level
  partitioning — no built-in cross-node vector index. This is the ceiling that
  pushes people toward dedicated engines or timescale/pgvectorscale.
- **Multitenancy caveat.** A shared approximate index lets one tenant's vectors
  degrade another tenant's recall and latency; isolate with list partitioning or
  separate tables.
- **License metadata.** GitHub reports the license as `NOASSERTION` because the
  repo ships the PostgreSQL License under a non-standard filename; it is a
  permissive, BSD-style license, not a restrictive one — the classifier is wrong,
  not the license.

## When to Use / When Not

**Use when:**
- Your embeddings belong to relational data you already store in Postgres and you
  want transactional consistency between them.
- Dataset is in the millions-to-tens-of-millions range and fits one primary.
- You want one system to operate, back up, and secure — not two.
- You need hybrid search (vector + Postgres full-text + SQL filters) in one query.

**Avoid when:**
- You are at hundreds of millions / billions of vectors and need distributed
  sharding and heavy quantization out of the box.
- Vector search is your entire workload with no relational component — a purpose-
  built engine will give better recall-at-latency per dollar.
- You need GPU-accelerated indexing or product quantization built in.

## Alternatives

- timescale/pgvectorscale — Postgres extension layered on pgvector adding
  StreamingDiskANN and statistical binary quantization; use when you want to stay
  in Postgres but push past pgvector's scaling limits.
- qdrant/qdrant — dedicated Rust vector database; use when vector search is the
  primary workload and you want quantization and filtering as first-class features.
- milvus-io/milvus — distributed, GPU-capable; use at billion-scale where
  horizontal sharding is mandatory.
- weaviate/weaviate — vector DB with built-in module ecosystem and hybrid search;
  use when you want an all-in-one search product rather than a Postgres add-on.
- elastic/elasticsearch — HNSW kNN alongside mature lexical search; use when you
  already run Elastic for full-text and want vectors in the same place.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021-04 | Initial release: `vector` type, IVFFlat, exact search[^1]. |
| 0.5.0 | 2023-08 | HNSW indexing; parallel IVFFlat builds[^3]. |
| 0.6.0 | 2024-01 | Parallel HNSW builds, faster index construction. |
| 0.7.0 | 2024-04 | `halfvec`, `sparsevec`, `bit` types; binary quantization; L1/Hamming/Jaccard[^2]. |
| 0.8.0 | 2024-10 | Iterative index scans for filtered queries; improved planner costing. |
| 0.8.5 | 2025 | Current maintenance release; Postgres 13–18 support[^4]. |

## References

[^1]: Andrew Kane, pgvector project. https://github.com/pgvector/pgvector
[^2]: pgvector README — types, distance functions, and quantization. https://github.com/pgvector/pgvector/blob/master/README.md
[^3]: pgvector CHANGELOG. https://github.com/pgvector/pgvector/blob/master/CHANGELOG.md
[^4]: pgvector releases. https://github.com/pgvector/pgvector/releases

## Tags

c, postgres, postgresql-extension, vector-search, embeddings, approximate-nearest-neighbor, hnsw, ivfflat, similarity-search, rag, database
