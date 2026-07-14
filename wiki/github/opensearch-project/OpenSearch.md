# opensearch-project/OpenSearch

> Apache-2.0 fork of Elasticsearch 7.10 — a distributed Lucene search and observability engine, now under the Linux Foundation.

[GitHub repo](https://github.com/opensearch-project/OpenSearch) ·
[Official website](https://opensearch.org) ·
[License: Apache-2.0](https://github.com/opensearch-project/OpenSearch/blob/main/LICENSE.txt)

## Overview

OpenSearch is a distributed, RESTful search and analytics engine built on Apache Lucene. It was forked from Elasticsearch 7.10.2 and Kibana 7.10.2 by AWS in 2021, after Elastic relicensed those projects from Apache-2.0 to the dual SSPL / Elastic License[^1]. This repository is the search engine core; the query UI lives separately in opensearch-project/OpenSearch-Dashboards (the Kibana fork). The 1.0 GA shipped in July 2021 and the project stayed Apache-2.0 throughout.

The defining fact about OpenSearch is its origin. Because it branched at 7.10.2, its data model, REST API, query DSL, and on-disk index format are near-identical to that Elasticsearch version, and it has diverged steadily since. That heritage is both the selling point (a permissively licensed drop-in for teams unwilling to accept Elastic's license terms) and the constant caveat (cross-fork compatibility erodes with every release on both sides). In September 2024 AWS transferred the project to the Linux Foundation, which created the OpenSearch Software Foundation to hold governance and the trademark[^2].

OpenSearch is used for full-text search, log and metrics analytics (the ELK/observability workload), and increasingly vector / semantic search for RAG pipelines via its k-NN and ml-commons plugins. Its main competitor is the very project it forked from.

## Getting Started

```bash
docker run -d -p 9200:9200 -p 9600:9600 \
  -e "discovery.type=single-node" \
  -e "OPENSEARCH_INITIAL_ADMIN_PASSWORD=<Strong-Password-123!>" \
  opensearchproject/opensearch:latest
```

```bash
# Index a document, then search it (security plugin on by default → -k + auth)
curl -k -u admin:'<Strong-Password-123!>' -X PUT "https://localhost:9200/movies/_doc/1" \
  -H 'Content-Type: application/json' \
  -d '{"title": "Dune", "year": 2021}'

curl -k -u admin:'<Strong-Password-123!>' -X GET "https://localhost:9200/movies/_search" \
  -H 'Content-Type: application/json' \
  -d '{"query": {"match": {"title": "dune"}}}'
```

Single-node Docker enables TLS and the security plugin by default; the initial admin password is mandatory as of 2.12. Production clusters are typically deployed via the Helm chart, the OpenSearch Kubernetes operator, or the tarball with `opensearch.yml`.

## Architecture / How It Works

A cluster is a set of JVM nodes. An index is split into **shards**, each of which is a self-contained Lucene index; shards have primary and replica copies distributed across data nodes. Documents are hashed to a shard by routing key, indexed into an in-memory buffer, made searchable on **refresh** (default 1s), and made durable via the **translog** (write-ahead log) flushed to Lucene segments. Segments are immutable and merged in the background.

Node roles separate concerns: the **cluster manager** node (renamed from Elasticsearch's "master" — the API accepts both `cluster_manager` and `master` for compatibility[^3]) owns cluster state; **data** nodes hold shards; **coordinating** and **ingest** nodes route and preprocess. Cluster-manager election needs a quorum of eligible nodes to avoid split-brain, so an odd count (typically 3) is standard.

Almost everything beyond core search is a **plugin** bundled into the distribution: `security` (TLS, RBAC, SAML/OIDC — the biggest behavioral difference from stock Elasticsearch, which gated security behind a paid tier at the fork point), `k-NN` (approximate vector search over Faiss / nmslib / Lucene HNSW), `ml-commons` and `neural-search` (model hosting and semantic search), `alerting`, `anomaly-detection`, `index-management` (ISM rollover / hot-warm-cold tiering), `sql`, and `observability`. These plugins are versioned and released together with the engine, which is why plugin/engine version skew is not a supported configuration.

OpenSearch 2.x runs on Lucene 9; OpenSearch 3.0 (2025) moved to Lucene 10 and a JDK 21 baseline, and added an experimental gRPC transport and pull-based ingestion[^4].

## Production Notes

**JVM heap.** Set heap to ~50% of node RAM and keep it under ~32 GB to preserve compressed object pointers; above that threshold the JVM loses compressed oops and effective heap can drop. The other half of RAM is left for the OS page cache, which is what actually makes Lucene fast.

**Shard sizing is the perennial footgun.** Too many small shards exhaust cluster-manager memory and slow every cluster-state update; oversized shards make recovery and rebalancing slow. The common guidance is 10–50 GB per shard and keeping total shard count proportional to heap. Over-sharding time-series indices (one index per day with default 5 shards) is the classic way to melt a logging cluster; use ISM rollover and shrink instead.

**Security plugin defaults changed.** Since 2.12 a strong initial admin password is required to boot the demo/default security config — automation that assumed `admin:admin` broke on upgrade. TLS on the transport layer is on by default, so misconfigured certs are a frequent first-boot failure.

**Elasticsearch cross-fork compatibility is a trap.** OpenSearch historically reported version `7.10.2` in a compatibility mode so that Elasticsearch clients would talk to it; later Elastic client libraries added product-checks that reject non-Elastic servers, so a stock ES client may refuse to connect. Snapshots restore only forward from the fork point — you can restore an Elasticsearch 6.x/7.x (≤7.10.2) snapshot into OpenSearch, but not a snapshot from a newer Elasticsearch. Plan migrations around the fork lineage, not marketing version numbers.

**Upgrades.** Rolling upgrades work within a major line and one major hop (1.x → 2.x, 2.x → 3.x); skipping a major generally requires reindex or snapshot/restore. Read the breaking-changes notes for renamed settings and removed types — the 1.x→2.x jump changed defaults around the `master`/`cluster_manager` terminology and dropped several deprecated APIs.

**Vector search cost.** k-NN with HNSW graphs is memory-hungry; the graphs are held largely off-heap and sizing them is separate from JVM heap math. Faiss vs. Lucene engine choice affects memory, filtering behavior, and recall.

## When to Use / When Not

**Use when:**
- You want a permissively (Apache-2.0) licensed Elasticsearch-compatible engine with security, alerting, and RBAC included at no cost.
- You're running log/metrics/observability analytics at scale (the ELK workload) and want to avoid Elastic licensing.
- You need built-in vector / hybrid search for RAG without bolting on a separate vector DB.
- You're on AWS and want managed compatibility (Amazon OpenSearch Service).

**Avoid when:**
- You need the newest Elasticsearch-only features (ES|QL, specific ML features) or Elastic Cloud's managed tooling — the forks have diverged.
- Your dataset is small and operationally simple — a JVM cluster with shard math is overkill vs. a lightweight engine.
- You cannot staff JVM tuning, shard planning, and cluster operations; the engine is powerful but not low-maintenance.
- You need a purpose-built vector database with the latest ANN research; dedicated vector stores iterate faster on that one axis.

## Alternatives

- elastic/elasticsearch — the upstream original; relicensed to add AGPL-3.0 in 2024. Use it instead when you want ES-only features, Elastic Cloud, or official Elastic support.
- apache/solr — the other mature Lucene search server. Use it instead when you want a long-stable, non-forked Lucene engine without the observability suite.
- meilisearch/meilisearch — lightweight typo-tolerant search. Use it instead for app/site search where a distributed JVM cluster is unwarranted.
- quickwit-oss/quickwit — log search on object storage (S3). Use it instead when log analytics on cheap decoupled storage matters more than sub-second freshness.
- typesense/typesense — in-memory instant-search engine. Use it instead for fast, simple faceted search with minimal operational overhead.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-04 | Project announced; fork of Elasticsearch/Kibana 7.10.2 after SSPL relicense[^1]. |
| 1.0 | 2021-07 | First GA under Apache-2.0. |
| 2.0 | 2022-05 | Lucene 9, `cluster_manager` terminology, security-by-default direction. |
| 2.12 | 2024-02 | Mandatory initial admin password for default security config. |
| — | 2024-09 | Governance transferred to the Linux Foundation (OpenSearch Software Foundation)[^2]. |
| 3.0 | 2025-05 | Lucene 10, JDK 21 baseline, experimental gRPC transport, pull-based ingestion[^4]. |

## References

[^1]: AWS, "Stepping up for a truly open source Elasticsearch" — 2021-01-21, and the OpenSearch launch — 2021-04. https://aws.amazon.com/blogs/opensource/stepping-up-for-a-truly-open-source-elasticsearch/
[^2]: OpenSearch, "AWS transfers OpenSearch to the OpenSearch Software Foundation" — 2024-09. https://opensearch.org/blog/opensearch-moves-to-foundation/
[^3]: OpenSearch documentation — cluster manager node and the `cluster_manager` / `master` compatibility. https://docs.opensearch.org/latest/tuning-your-cluster/cluster/
[^4]: OpenSearch, "Get started with OpenSearch 3.0" — 2025. https://opensearch.org/blog/get-started-with-opensearch-3-0/

## Tags

java, search-engine, lucene, distributed-systems, full-text-search, vector-search, observability, log-analytics, elasticsearch-fork, apache-2.0, rest-api
