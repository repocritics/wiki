# quickwit-oss/quickwit

> Sub-second full-text search directly on object storage (S3/GCS/Azure), purpose-built for logs and traces rather than general-purpose search.

[GitHub repo](https://github.com/quickwit-oss/quickwit) ·
[Official website](https://quickwit.io) ·
[License: Apache-2.0](https://github.com/quickwit-oss/quickwit/blob/main/LICENSE)

## Overview

Quickwit is a distributed search engine written in Rust, built on the Tantivy full-text library (also authored by Quickwit's founder Paul Masurel)[^1]. Its defining bet is architectural: instead of keeping indexes on local disk like Elasticsearch, Quickwit stores them as immutable "split" files on cloud object storage and queries them there directly. Compute and storage are decoupled, indexers and searchers are stateless, and the cluster scales out in seconds. The tradeoff is latency measured in the hundreds of milliseconds rather than single-digit milliseconds — acceptable for observability, disqualifying for user-facing search.

The intended audience is teams running large log and trace volumes who find Elasticsearch's hot-storage cost model painful. Quickwit ships an Elasticsearch-compatible ingest and query API subset, is OTEL-native for logs and traces, and plugs into Jaeger and Grafana as a backend[^2]. Metrics are on the roadmap but not shipped. It is explicitly not a general-purpose search engine — there is no document mutability, relevance tuning is shallower than Elasticsearch's, and deletes are a batch operation, not a per-document one.

The most material fact for anyone adopting Quickwit today is governance: Quickwit Inc., the company behind the project, was acquired by Datadog in late 2024[^3]. The repository remains Apache-2.0 and actively committed, but the original commercial roadmap effectively ended, and long-term direction now depends on Datadog's internal priorities rather than an independent OSS vendor.

## Getting Started

```bash
# Download and run a single-node instance
curl -L https://install.quickwit.io | sh
cd quickwit-*/
./quickwit run
```

```bash
# Create an index from a doc-mapping config, then ingest and search
./quickwit index create --index-config ./config/tutorials/hdfs-logs/index-config.yaml
./quickwit index ingest --index hdfs-logs --input-path ./hdfs-logs.json
./quickwit index search --index hdfs-logs --query "severity:ERROR"
```

```bash
# Elasticsearch-compatible ingest endpoint — point Vector / Fluent Bit here
curl -XPOST http://localhost:7280/api/v1/hdfs-logs/ingest \
  --data-binary @docs.ndjson
```

## Architecture / How It Works

Quickwit is a single binary that runs one or more services: **indexers**, **searchers**, a **metastore**, a **control plane**, and a **janitor** (retention, GC, delete tasks). A node can run all of them or a subset, which is how the same build serves both a laptop demo and a sharded cluster.

- **Splits.** An index is a set of splits — each split is a self-contained Tantivy index, immutable once written, uploaded to object storage. Every split embeds a "hotcache," a small index-of-the-index that lets a searcher answer a query with a handful of ranged GET requests instead of scanning the whole file. This is what makes search-on-S3 sub-second.
- **Metastore.** The metastore tracks index metadata and the list of splits. It is backed either by a file (JSON on the object store) or by PostgreSQL. The file backend assumes a single writer; anything multi-node or HA requires PostgreSQL[^4].
- **Statelessness.** Because splits live on object storage and state lives in the metastore, indexers and searchers hold no durable local data. A searcher can be killed and replaced without recovery; you scale reads and writes independently.
- **Cluster membership** uses chitchat, Quickwit's SWIM-style gossip library, over which the control plane assigns indexing and search work. Inter-node traffic is gRPC; the user-facing API is REST.
- **Ingestion** comes from Kafka, Kinesis, or Pulsar sources, or the push-based ingest API. Documents are batched, indexed into a split, committed, and merged in the background — small splits from frequent commits are periodically compacted into larger ones.

The whole design co-evolves with Tantivy; new query and aggregation capabilities in Quickwit generally track Tantivy features underneath.

## Production Notes

**It is an append-mostly store, not a database.** Documents are immutable. There are no partial updates. "Deletes" are asynchronous delete tasks that rewrite affected splits — designed for GDPR erasure, not routine mutation, and expensive at volume. If your data changes after ingestion, Quickwit is the wrong tool.

**Latency profile.** Search reads from object storage, so first-query latency is dominated by S3 round-trips. Expect hundreds of milliseconds, and design for concurrent low-QPS analytical queries rather than high-QPS point lookups. Caching (hotcache plus searcher-side caches) helps repeated queries but does not turn it into a millisecond-latency engine.

**HA has an asterisk.** Per the project's own documentation, indexing is highly available only when the source is Kafka; the push ingest API is not HA in the same way. Search HA requires multiple searchers and a PostgreSQL metastore. The file-backed metastore is single-writer and will corrupt or lose writes under concurrent access — do not run it multi-node.

**Commit granularity is a footgun.** Committing too frequently produces many tiny splits, which increases the number of object-storage requests per search until background merges catch up. Tune commit timeout and merge policy for your ingest rate.

**Elasticsearch compatibility is a subset.** The ingest API and the most common query DSL and aggregations are supported, enough to migrate log shippers and many dashboards, but it is not a drop-in for every Elasticsearch feature. Verify the specific endpoints, queries, and aggregations you depend on against the compatibility docs before committing to a migration[^2].

**Upgrades touch on-disk formats.** Split and metastore formats have changed across minor versions; historically Quickwit has provided a migration path but not always guaranteed skipping intermediate versions. Read release notes and back up the metastore before upgrading.

## When to Use / When Not

**Use when:**
- You ingest large log or trace volumes and want object-storage economics instead of Elasticsearch hot-node costs.
- You need an OTEL/Jaeger/Grafana backend with full-text search over observability data.
- You want Elasticsearch-compatible ingest and query for logs without running Elasticsearch.
- Your data is write-once, search-later, and tolerant of sub-second (not sub-millisecond) latency.

**Avoid when:**
- You need user-facing search: product catalogs, autocomplete, or anything latency-critical.
- Your documents get updated or deleted individually and often.
- You need a metrics store today (not yet supported).
- You require the full breadth of Elasticsearch relevance tuning, query features, or ecosystem plugins.

## Alternatives

- elastic/elasticsearch — use when you need general-purpose search, mutable documents, deep relevance tuning, or low-latency serving and can pay for hot storage.
- opensearch-project/OpenSearch — use when you want Elasticsearch-class features under an Apache-2.0 license with open governance.
- grafana/loki — use when you want the cheapest possible log storage and label-based filtering is enough, without full-text indexing of message bodies.
- ClickHouse/ClickHouse — use when observability is analytics- and metrics-heavy rather than full-text-search-heavy.
- quickwit-oss/tantivy — use when you want to embed full-text search inside your own application instead of running a search service.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2021 | First public release; search on object storage built on Tantivy[^1]. |
| 0.3 | 2022 | Kafka source, distributed search maturing. |
| 0.5 | 2023 | OTEL-native logs/traces, Jaeger integration. |
| 0.6 | 2023 | Elasticsearch-compatible API expansion. |
| 0.7 | 2024 | Ingest V2 work, PostgreSQL metastore improvements. |
| 0.8 | 2024 | Current line per README; performance and compatibility gains[^5]. |

Exact release dates for older minor versions were not verified against a live source and are given at year granularity.

## References

[^1]: Quickwit is built on the Tantivy search library. https://github.com/quickwit-oss/tantivy
[^2]: Quickwit Elasticsearch-compatible API reference. https://quickwit.io/docs/reference/es_compatible_api
[^3]: Datadog announced its acquisition of Quickwit in late 2024. https://www.datadoghq.com/blog/engineering/datadog-acquires-quickwit/
[^4]: Quickwit metastore configuration (file vs PostgreSQL). https://quickwit.io/docs/overview/architecture
[^5]: Quickwit 0.8 release announcement. https://quickwit.io/blog/quickwit-0.8

## Tags

rust, search-engine, observability, log-management, distributed-tracing, cloud-native, object-storage, elasticsearch-compatible, tantivy, otel, full-text-search
