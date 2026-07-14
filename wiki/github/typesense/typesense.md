# typesense/typesense

> An in-memory, typo-tolerant search engine positioned as an open-source Algolia and an easier-to-operate Elasticsearch.

[GitHub repo](https://github.com/typesense/typesense) ·
[Official website](https://typesense.org) ·
[License: GPL-3.0](https://github.com/typesense/typesense/blob/v31/LICENSE)

## Overview

Typesense is a search engine written in C++, first published in 2017[^1]. It ships as a single self-contained binary with no runtime dependencies: you download it, point it at a data directory, and talk to it over HTTP/JSON. The pitch is deliberately narrow — instant, typo-tolerant, faceted search with smart defaults — rather than the general-purpose datastore-and-analytics scope of Elasticsearch. In practice it competes on two fronts at once: against the proprietary hosted Algolia (on price and self-hostability) and against Elasticsearch (on operational simplicity)[^2].

The defining architectural choice is that the searchable index lives entirely in RAM. This is what makes latency low and predictable, and it is also the single fact that governs cost, capacity planning, and the ceiling on how large a dataset you can serve. Typesense does not shard: every node in a cluster holds a full copy of the data, so the working set must fit in the memory of one machine. Clustering buys you high availability and read throughput, not horizontal data volume[^3].

Over time the surface has grown well past keyword search — vector search, built-in embedding generation, hybrid/semantic search, conversational RAG, and geo search are all in-tree. That breadth is real, but the engine's center of gravity remains fast lexical search with typo tolerance, and most production deployments use it for exactly that.

## Getting Started

```bash
# Run the server (Docker). Replace the api-key with your own.
docker run -p 8108:8108 -v /tmp/typesense-data:/data \
  typesense/typesense:29.0 --data-dir /data --api-key=xyz
```

```python
import typesense

client = typesense.Client({
    "api_key": "xyz",
    "nodes": [{"host": "localhost", "port": "8108", "protocol": "http"}],
    "connection_timeout_seconds": 2,
})

client.collections.create({
    "name": "companies",
    "fields": [
        {"name": "company_name", "type": "string"},
        {"name": "num_employees", "type": "int32"},
        {"name": "country", "type": "string", "facet": True},
    ],
    "default_sorting_field": "num_employees",
})

client.collections["companies"].documents.create(
    {"id": "124", "company_name": "Stark Industries",
     "num_employees": 5215, "country": "USA"}
)

# "stork" is a typo — Typesense still finds "Stark Industries".
client.collections["companies"].documents.search(
    {"q": "stork", "query_by": "company_name", "sort_by": "num_employees:desc"}
)
```

## Architecture / How It Works

Typesense is a single native binary. Incoming writes are appended to an on-disk write-ahead log (backed by RocksDB) for durability, and the in-memory indexes are rebuilt from that log on startup. Reads never touch disk on the hot path — they are served from RAM-resident structures. This split is why a cold start on a large collection can take minutes (the index is reconstructed from the WAL) even though steady-state query latency is single- to low-double-digit milliseconds.

The lexical index is built for prefix and fuzzy matching, which is what powers typo tolerance and type-ahead out of the box: a query is expanded within a bounded edit distance and ranked by a fixed set of factors (text match, then any sort/ranking fields you configure). Unlike Algolia, most of these knobs — which fields to search, facet, group, and how to rank — are supplied per query rather than baked into the index at creation time, so a single index can serve many sort orders without duplication[^2].

Clustering uses the Raft consensus protocol (via the braft library) for leader election and log replication. A typical highly-available deployment is three nodes; one leader takes writes and replicates them to followers, and any node can serve reads. Because there is no sharding, adding nodes scales read capacity and availability, not dataset size.

Vector search is implemented with HNSW (approximate nearest-neighbor) graphs held alongside the keyword index, which is what allows Typesense to run keyword, vector, and hybrid queries through one API. Optional embedding generation (built-in models such as E5/S-BERT, or remote providers like OpenAI) runs inside the server so you can send raw JSON and get semantic search without a separate embedding pipeline.

## Production Notes

- **RAM is the budget.** Plan capacity by index size in memory, not on-disk document size — indexed data is typically several times larger in RAM than the raw JSON. Because there is no sharding, your dataset ceiling is one node's memory; there is no built-in way to split a collection across machines[^3].
- **Cold starts and restarts are not free.** On restart the index rebuilds from the WAL. For large collections this can take minutes, during which the node is not serving. Rolling restarts across an HA cluster are the standard mitigation.
- **Upgrades are binary swaps, but read the notes.** The project advertises upgrades as "swap the binary and restart," which holds for most releases, but index-format or default-behavior changes do appear across major versions — test on a replica first. Version numbering changed from a `0.x` scheme to a plain integer scheme (0.25 → 26.0) in 2024, which can confuse tooling and pinned tags[^4].
- **API-key surface.** Access control is via API keys, including scoped keys that restrict results by filter for multi-tenant apps. Treat the bootstrap `--api-key` as an admin credential; do not ship it to browsers — generate scoped search-only keys for client-side use.
- **Analytics and personalization are thin.** Compared with Algolia, server-side search analytics and personalization are limited; teams generally instrument the client and ship metrics to their own analytics stack.
- **License.** The server is GPL-3.0, deliberately not AGPL — the maintainers reason that Typesense runs as a standalone daemon rather than being linked into your code, so GPL does not reach across the network boundary[^5]. This is comfortable for most operators but worth a legal read if you fork and redistribute a modified server.

## When to Use / When Not

**Use when:**
- You want instant, typo-tolerant, faceted search with minimal tuning and a clean JSON API.
- You'd rather self-host (or pay a flat cluster price) than meter records and operations on Algolia.
- Your dataset fits comfortably in RAM on a single node and latency predictability matters.
- You want keyword and vector/semantic search behind one engine without wiring a separate vector DB.

**Avoid when:**
- Your corpus is too large to fit in a single node's memory — there is no sharding to grow into.
- You need log/observability-scale ingestion, complex aggregations, or petabyte storage — that is Elasticsearch/OpenSearch territory.
- You need rich built-in search analytics or personalization today.
- GPL-3.0 on a modified-and-redistributed server binary is a problem for your distribution model.

## Alternatives

- meilisearch/meilisearch — Rust engine with a very similar instant-search DX and MIT license; choose it when you want comparable simplicity outside the C++/GPL stack.
- elastic/elasticsearch — reach for it when you need sharding, heavy aggregations, and log/analytics workloads rather than curated site search.
- opensearch-project/OpenSearch — Apache-2.0 Elasticsearch fork; use it when you want ES compatibility and scale without ES licensing concerns.
- manticoresoftware/manticoresearch — SQL-first, supports disk-based indexes; better when the dataset is too large to keep fully in RAM.
- qdrant/qdrant — dedicated vector database; prefer it when the workload is pure semantic/vector search rather than keyword-first with vectors bolted on.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial repo | 2017-01 | First public commit; C++ in-memory search engine[^1]. |
| 0.x series | 2018–2023 | Long-running `0.x` line; faceting, geo, filtering, Raft clustering, vector search added incrementally. |
| 26.0 | 2024 | Version scheme dropped the `0.` prefix (0.25 → 26.0)[^4]. |
| 27–29 | 2024–2026 | Semantic/hybrid search, conversational RAG, natural-language and image/voice search matured. |
| v31 (dev) | 2026 | Active development branch as of mid-2026. |

## References

[^1]: Typesense GitHub repository (created 2017-01-18). https://github.com/typesense/typesense
[^2]: Typesense README — "How does this differ from Algolia / Elasticsearch?" https://github.com/typesense/typesense#faq
[^3]: Typesense docs — high availability and cluster operations (each node holds a full copy; no sharding). https://typesense.org/docs/
[^4]: Typesense downloads and release notes (version-scheme change to integer majors). https://typesense.org/downloads
[^5]: Typesense README — "Why the GPL license?" (deliberate GPL-3.0 over AGPL). https://github.com/typesense/typesense#faq

## Tags

c-plus-plus, search-engine, full-text-search, typo-tolerance, vector-search, semantic-search, in-memory, faceted-search, self-hosted, algolia-alternative, elasticsearch-alternative
