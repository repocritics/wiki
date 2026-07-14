# elastic/elasticsearch

> Distributed search and analytics engine built on Apache Lucene, driven entirely over a JSON REST API.

[GitHub repo](https://github.com/elastic/elasticsearch) ·
[Official website](https://www.elastic.co/products/elasticsearch) ·
[License: Elastic-2.0 / SSPL-1.0 / AGPL-3.0 tri-license](https://github.com/elastic/elasticsearch/blob/main/LICENSE.txt)

## Overview

Elasticsearch is a distributed document store and search engine that wraps Apache Lucene with a clustering layer, a JSON query language, and a REST API. Shay Banon released it in 2010; the company Elastic (formerly Elasticsearch BV) was built around it and its sibling projects Kibana, Logstash, and Beats — collectively the "Elastic Stack" (once "ELK")[^1]. It is one of the most widely deployed pieces of infrastructure in existence: log analytics, full-text search, application performance monitoring, security analytics (SIEM), and, since 8.0, dense-vector / kNN search for RAG and semantic retrieval.

The defining characteristic is that Elasticsearch is Lucene made distributed and operable at scale. A single Lucene index becomes a **shard**; an Elasticsearch index is a collection of primary and replica shards spread across nodes. Everything else — cluster coordination, replication, the query DSL, aggregations, ingest pipelines — is machinery layered on top so that Lucene's per-segment inverted indexes can be sharded, replicated, and queried as one logical store.

The most consequential fact about the project is its **licensing history**. Elasticsearch shipped under Apache 2.0 until 7.11 (February 2021), when Elastic relicensed to a dual SSPL 1.0 / Elastic License 2.0 model to counter AWS offering it as a managed service[^2]. AWS forked the 7.10 codebase into OpenSearch. In late 2024 Elastic added AGPL 3.0 as a third option, restoring an OSI-approved license[^3]. Which "Elasticsearch" you mean now depends on the version and the license you accept.

## Getting Started

The `start-local` script brings up Elasticsearch + Kibana in Docker for development (not production)[^4]:

```bash
curl -fsSL https://elastic.co/start-local | sh
```

Index and search documents over REST (the entire API surface is HTTP + JSON):

```bash
# Index a document — the index is auto-created on first write
curl -u elastic:$ES_LOCAL_PASSWORD -X POST http://localhost:9200/customer/_doc/1 \
  -H 'Content-Type: application/json' \
  -d '{ "firstname": "Jennifer", "lastname": "Walters" }'

# Full-text query via the Query DSL
curl -u elastic:$ES_LOCAL_PASSWORD -X GET http://localhost:9200/customer/_search \
  -H 'Content-Type: application/json' \
  -d '{ "query": { "match": { "firstname": "Jennifer" } } }'
```

For bulk loads use the `_bulk` API with newline-delimited JSON (NDJSON); single-document indexing does not scale for ingest.

## Architecture / How It Works

**Storage.** Each shard is a full Lucene index: immutable **segments** of an inverted index, plus doc values (columnar, for sorting/aggregations), stored fields, and optional `dense_vector` HNSW graphs for kNN. Writes go to an in-memory buffer plus a **translog** (write-ahead log for durability). A **refresh** (default every 1s) makes new docs searchable by opening a new segment; a **flush** fsyncs to Lucene and trims the translog; background **merges** compact small segments into larger ones. This is why Elasticsearch is "near real-time" rather than immediately consistent — there is a refresh interval between write and visibility.

**Distribution.** An index is split into a fixed number of **primary shards** (set at creation, not changeable without reindex) plus configurable **replica shards**. Nodes take roles: master-eligible (cluster state), data (hold shards), ingest (run pipelines), coordinating (route/merge). A search fans out to one copy of every shard (the query phase), then gathers and re-ranks the top hits (the fetch phase) — the classic scatter/gather. Since 7.0 cluster coordination uses a Raft-like consensus (replacing the older Zen discovery), which removed the manual `minimum_master_nodes` split-brain footgun[^5].

**Query surface.** Three overlapping languages: the JSON **Query DSL** (original, most complete), **ES|QL** — a piped, SQL-like query language introduced in the 8.x line for search + aggregation over one syntax, and SQL/EQL for specific workloads. Aggregations run the analytics side (histograms, cardinality, percentiles) and are the reason Elasticsearch doubles as an analytics store.

**Mappings.** Each index has a schema (mapping) that assigns each field an analyzer and Lucene type. Dynamic mapping infers types on first write — convenient, and a common source of production incidents when an unexpected value locks a field into the wrong type.

## Production Notes

The differentiator between "works in a demo" and "works at scale" is operational discipline. The recurring footguns:

- **JVM heap.** Set the heap to no more than 50% of RAM and keep it under ~30–31 GB so the JVM keeps compressed object pointers. The other half of RAM must be left free for the OS filesystem cache — Lucene mmaps its segments and relies on page cache for speed. Over-allocating heap is the classic mistake that makes a cluster slower, not faster.
- **Shard sizing.** Each shard carries fixed overhead (heap, file handles, cluster-state bookkeeping). Target roughly 10–50 GB per shard and a bounded shard count per GB of heap. **Oversharding** — thousands of tiny shards from per-day indices on low-volume data streams — is the most common cause of cluster instability. Use ILM / rollover to size indices, not a fixed per-day scheme.
- **Primary shard count is immutable.** You choose it at index creation; changing it means reindex or `_split`/`_shrink`. Over-provisioning "just in case" causes oversharding; under-provisioning caps write throughput. There is no free lunch.
- **Mapping explosion.** Dynamic mapping plus arbitrary JSON keys can blow past the 1,000-field limit and bloat cluster state. Disable dynamic mapping or use `flattened` fields for user-controlled data.
- **Deep pagination.** `from`/`size` deep paging re-collects and re-sorts on every shard and is O(from+size) per shard. Use `search_after` with a Point-in-Time (PIT) for cursoring large result sets.
- **Circuit breakers and aggregations.** High-cardinality aggregations and large `terms` buckets can trip the request circuit breaker or, worse, cause GC pressure. Test aggregations against production-scale cardinality, not sample data.
- **Upgrades.** Rolling upgrades are supported but only one major version at a time — you cannot skip a major, and Lucene only reads indices from one major version back, so very old indices must be reindexed before a jump. Read the breaking-changes and deprecation logs before every major.
- **Backups.** The only supported backup is the snapshot/restore API to a repository (S3, GCS, shared FS). Copying the data directory of a live node is not a backup.

## When to Use / When Not

**Use when:**
- You need full-text relevance search (BM25, analyzers, highlighting) over large corpora.
- You're doing log/metric/trace analytics or SIEM at volume and want aggregations + time-based retention (ILM, data streams).
- You need hybrid search: keyword BM25 combined with dense-vector kNN for RAG.
- You want horizontal scale-out and replication without building sharding yourself.

**Avoid when:**
- You need a system of record with transactions and strong consistency — Elasticsearch is near-real-time and eventually consistent; keep the authoritative copy elsewhere.
- Your dataset is small and search needs are modest — a Postgres full-text index or a lighter engine is far less to operate.
- You cannot staff the operational burden (heap tuning, shard strategy, JVM/GC, upgrades).
- License terms matter and you need a permissive OSI license across all versions — evaluate the tri-license carefully or use OpenSearch.

## Alternatives

- opensearch-project/OpenSearch — the AWS Apache-2.0 fork of the 7.10 code; use when you need a fully permissive license or AWS-managed OpenSearch Service.
- apache/solr — the other mature Lucene-based engine; use when you want Apache-licensed search without Elastic's ecosystem or license questions.
- meilisearch/meilisearch — use when you want instant typo-tolerant search over modest datasets with minimal ops, not petabyte analytics.
- quickwit-oss/quickwit — use when log/trace search on cheap object storage matters more than sub-second real-time indexing.
- typesense/typesense — use when you want a small, fast, developer-friendly search API and don't need Elasticsearch's aggregation depth.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.4 | 2010-02 | First public release by Shay Banon, on Lucene[^1]. |
| 1.0 | 2014-02 | Snapshot/restore, aggregations foundation. |
| 2.0 | 2015-10 | Lucene 5, pipeline aggregations. |
| 5.0 | 2016-10 | Version aligned across the Elastic Stack; ingest nodes. |
| 6.0 | 2017-11 | Single mapping type per index, rolling-upgrade improvements. |
| 7.0 | 2019-04 | New Raft-like cluster coordination; one primary shard default[^5]. |
| 7.11 | 2021-02 | Relicensed Apache 2.0 → SSPL / Elastic License; AWS forks OpenSearch[^2]. |
| 8.0 | 2022-02 | Security on by default; native kNN dense-vector search. |
| 8.11 | 2023-11 | ES|QL piped query language introduced (preview). |
| 8.16 | 2024 | AGPL 3.0 added as a third license option[^3]. |

## References

[^1]: Elastic, "The history of Elasticsearch" / company about page. https://www.elastic.co/about/history-of-elasticsearch
[^2]: Elastic, "Doubling down on open, Part II" — 2021-01-19. https://www.elastic.co/blog/licensing-change
[^3]: Shay Banon, "Elasticsearch is Open Source, Again" — 2024-08-29. https://www.elastic.co/blog/elasticsearch-is-open-source-again
[^4]: Elasticsearch README / `start-local` quickstart. https://github.com/elastic/start-local
[^5]: Elasticsearch reference, "Cluster coordination". https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery.html

## Tags

search-engine, java, lucene, distributed-systems, full-text-search, vector-search, analytics, log-management, rest-api, elastic-stack, information-retrieval
