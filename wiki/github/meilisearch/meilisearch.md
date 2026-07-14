# meilisearch/meilisearch

> A Rust full-text and hybrid search engine that ships as a single binary and optimizes for instant, typo-tolerant, user-facing search.

[GitHub repo](https://github.com/meilisearch/meilisearch) ·
[Official website](https://www.meilisearch.com) ·
License: MIT (core) / BSL-1.1 (Enterprise Edition)

## Overview

Meilisearch is a search engine written in Rust, first published in 2018[^1], aimed squarely at end-user "search-as-you-type" experiences rather than at log analytics or general-purpose document stores. Its defining pitch is that relevant, typo-tolerant results should be available in a single self-hosted binary with sane defaults, no query DSL to learn, and sub-50ms responses out of the box. It reached a stable 1.0 in February 2023[^2] after years of pre-1.0 iteration.

The engine's design center is a small, opinionated relevancy model: a fixed pipeline of ranking rules (words, typo, proximity, attribute, sort, exactness) that you reorder rather than a scoring language you compose. This makes Meilisearch fast to adopt and predictable, but means it deliberately does not compete with Elasticsearch/OpenSearch on aggregations, complex boolean scoring, or petabyte-scale log ingestion. Since roughly 2023 it has added vector and hybrid (keyword + semantic) search on top of the same store[^3].

The most important recent tension is licensing. Meilisearch is no longer uniformly MIT: the core "Community Edition" stays MIT, but an "Enterprise Edition" gates features like sharding and S3-streaming snapshots behind a commercial license or the Business Source License 1.1[^4]. GitHub reports the repository license as `NOASSERTION` because of this split. Evaluate which features you need against which edition licenses them before building on it.

## Getting Started

```bash
# Run the single binary (Docker)
docker run -it --rm -p 7700:7700 \
  -e MEILI_MASTER_KEY="aSampleMasterKey" \
  -v $(pwd)/meili_data:/meili_data \
  getmeili/meilisearch:latest
```

```bash
# Add documents to an index (auto-creates it)
curl -X POST 'http://localhost:7700/indexes/movies/documents' \
  -H 'Authorization: Bearer aSampleMasterKey' \
  -H 'Content-Type: application/json' \
  --data-binary '[{ "id": 1, "title": "Interstellar", "genre": "sci-fi" }]'

# Search (typo tolerance is on by default)
curl 'http://localhost:7700/indexes/movies/search' \
  -H 'Authorization: Bearer aSampleMasterKey' \
  -H 'Content-Type: application/json' \
  --data-binary '{ "q": "interstelar" }'
```

Indexing is asynchronous: the documents call returns a `taskUid`, and the write is applied by a background task. Poll `/tasks/{uid}` to confirm the document is searchable.

## Architecture / How It Works

Meilisearch is a thin HTTP API (`meilisearch-http`) over a core search library historically named `milli`, which owns indexing, the ranking pipeline, and query execution.

- **Storage: LMDB via `heed`.** All index data lives in a memory-mapped LMDB key-value store. LMDB gives lock-free concurrent readers with a single writer and copy-on-write B-trees, so search reads never block on writes[^5]. The flip side is LMDB's single-writer model: all indexing serializes through one writer transaction.
- **Indexing is a batched, single-writer pipeline.** Write tasks (document additions, settings changes) are enqueued in a task queue and processed by a background scheduler that batches compatible tasks into one transaction. Large ingests use `grenad`-based external sorting to build the inverted index and other databases, then commit atomically.
- **Tokenization via `charabia`,** which handles Unicode normalization and script-aware segmentation, including CJK (Chinese/Japanese) word splitting where whitespace tokenization fails.
- **Ranking is bucket sort, not a float score.** Candidate documents pass through the ordered ranking rules; each rule partitions the current bucket into finer buckets. This is why relevancy is tuned by reordering rules rather than weighting a scoring formula.
- **Vector / hybrid search** stores embeddings in an LMDB-backed approximate-nearest-neighbor structure (`arroy`)[^3]. Meilisearch can call out to an embedder (e.g. OpenAI-compatible endpoints) or accept user-provided vectors, and `hybrid` queries blend semantic and keyword rankings by a `semanticRatio`.

The whole system is one binary with one on-disk database directory. There is no separate coordinator, broker, or JVM. That simplicity is the product.

## Production Notes

- **RAM and the page cache decide performance.** LMDB memory-maps the database; hot search performance depends on the working set fitting in OS page cache. A dataset much larger than available RAM will fault to disk and slow tail latencies. Size the box to the index, not just to the process's resident memory.
- **Single writer = bounded write throughput.** Because indexing serializes through one LMDB writer, heavy or bursty ingestion is best submitted in batches; many small document POSTs create task-queue churn. Search stays fast during indexing, but write latency is not horizontally scalable within one instance.
- **The `NOASSERTION` license split has operational teeth.** Sharding, replication topologies, and S3-streaming snapshots are Enterprise Edition features under BSL-1.1 / commercial terms and are "not allowed in production without a commercial agreement"[^4]. On the MIT Community Edition, a single instance has no native clustering; the common HA pattern is redundant instances each fed the same writes behind a load balancer.
- **Upgrades historically required a dump-and-reimport.** Across incompatible internal database versions, the classic upgrade path was export a dump on the old version and import on the new one — an operationally heavy, potentially long step for large indexes. Newer releases added an in-place ("dumpless") upgrade path[^6], but verify the exact upgrade mechanics for your source and target versions before a production bump.
- **Disk sizing looks alarming.** LMDB pre-reserves a large max map size, so the database file can appear very large (sparse) on disk even when little is used; monitor actual usage, not the apparent file size.
- **Telemetry is on by default.** Anonymized usage data is collected unless you disable it via `--no-analytics` / `MEILI_NO_ANALYTICS`[^7].
- **Secure the master key.** Running without a master key leaves the instance unauthenticated; the master key derives the API keys used for per-index and multi-tenant (tenant-token) access control.

## When to Use / When Not

**Use when:**
- You want instant, typo-tolerant, faceted search for a site or app with minimal tuning.
- You want one self-hosted binary with no JVM, cluster, or query DSL to operate.
- You need keyword and semantic/hybrid search over the same dataset without wiring a separate vector database.

**Avoid when:**
- You need heavy aggregations, analytics, or log/observability search at scale — reach for Elasticsearch/OpenSearch or Quickwit.
- Your dataset vastly exceeds available RAM and you need it all hot; the page-cache dependency will bite.
- You need native, MIT-licensed horizontal write scaling — that now lives in the BSL-licensed Enterprise Edition.
- You need a mature scoring/relevance language for bespoke ranking math rather than a fixed rule pipeline.

## Alternatives

- typesense/typesense — closest peer; C++ instant-search engine with a similar out-of-the-box relevancy story. Prefer it if you want an Apache-2.0 license without an enterprise-edition split.
- elastic/elasticsearch — use instead when you need aggregations, complex boolean scoring, or log/analytics workloads at large scale.
- quickwit-oss/quickwit — use for append-heavy log and trace search on object storage (S3) rather than user-facing autocomplete.
- valeriansaliou/sonic — use when you want an extremely lightweight, low-RAM identifier index and will fetch documents from your own store.
- paradedb/paradedb — use when you want full-text and hybrid search to live inside PostgreSQL instead of a separate service.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2018 | First public release as a Rust search engine[^1]. |
| 0.21 | 2021 | Major indexer rewrite around the `milli` core; large indexing-speed gains. |
| 1.0 | 2023-02 | First stable release; frozen API and on-disk format guarantees[^2]. |
| 1.3 | 2023 | Vector store and experimental semantic search introduced[^3]. |
| 1.x | 2024 | Hybrid keyword+semantic search matured; embedder integrations expanded. |
| 1.x | 2025 | In-place ("dumpless") upgrade path added to ease version migrations[^6]. |
| — | 2026 | Community Edition (MIT) / Enterprise Edition (BSL-1.1) split; repo license shows `NOASSERTION`[^4]. |

## References

[^1]: Meilisearch repository, created 2018-04-23. https://github.com/meilisearch/meilisearch
[^2]: Meilisearch blog, "Announcing Meilisearch 1.0" — February 2023. https://blog.meilisearch.com/meilisearch-1-0/
[^3]: Meilisearch docs, vector / hybrid search. https://www.meilisearch.com/docs/learn/ai_powered_search/getting_started_with_ai_search
[^4]: README "Editions & Licensing"; Business Source License 1.1. https://github.com/meilisearch/meilisearch#-editions--licensing and https://mariadb.com/bsl11
[^5]: LMDB (Lightning Memory-Mapped Database), accessed in Rust via `heed`. https://github.com/meilisearch/heed
[^6]: Meilisearch docs, updating / dumpless upgrade. https://www.meilisearch.com/docs/learn/update_and_migration/updating
[^7]: Meilisearch docs, telemetry and how to disable it. https://www.meilisearch.com/docs/learn/what_is_meilisearch/telemetry

## Tags

rust, search-engine, full-text-search, vector-search, hybrid-search, typo-tolerance, faceted-search, lmdb, self-hosted, semantic-search, instant-search
