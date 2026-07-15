# apache/solr

> A search server built on Apache Lucene — inverted-index full-text search, faceting, and (since 9.0) dense-vector KNN, exposed over HTTP.

[GitHub repo](https://github.com/apache/solr) ·
[Official website](https://solr.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/solr/blob/main/LICENSE.txt)

## Overview

Solr is a standalone search server that wraps the Apache Lucene indexing library behind an HTTP API. It was created by Yonik Seeley at CNET in 2004, donated to the Apache Software Foundation in 2006, and merged into a combined `lucene-solr` repository in 2010[^1]. In 2021 the two projects split their source trees again; this `apache/solr` repository (created 2021-02) holds Solr from 9.0 onward, while the library lives at `apache/lucene`. That lineage matters: almost every capability and limitation of Solr traces back to Lucene, and Solr's job is to add distribution, schema management, request handling, and operational tooling on top.

Solr's core is a mature, fully Apache-2.0-licensed alternative to Elasticsearch/OpenSearch, both of which share the same Lucene foundation. It is most entrenched in enterprise text search, e-commerce catalog and faceted search, digital libraries, and content platforms — workloads where relevance tuning, faceting, and structured filtering matter more than log-analytics dashboards. Since 9.0 it also indexes dense vectors and serves approximate KNN queries via Lucene's HNSW implementation, giving it a hybrid lexical + vector story[^2].

The defining tension is age and operational weight. Solr is deeply capable and battle-tested, but its distributed mode (SolrCloud) depends on an external Apache ZooKeeper ensemble, its configuration surface is large, and the star/fork counts (~1.6k stars, ~840 forks as of 2026) understate its production footprint — much of its user base predates GitHub-centric discovery and lives on Apache mailing lists and JIRA. It is actively maintained (commits within the last day at time of writing) but its center of gravity is stability, not novelty.

## Getting Started

```bash
# Docker is the quickest path to a running instance
docker run -d -p 8983:8983 --name solr solr:9

# Create a collection (single-node "core" here)
docker exec solr bin/solr create -c books
```

Index a document and query it over HTTP:

```bash
# Index one JSON document
curl -X POST 'http://localhost:8983/solr/books/update?commit=true' \
  -H 'Content-Type: application/json' \
  -d '[{"id":"1","title_t":"The Left Hand of Darkness","author_s":"Le Guin"}]'

# Full-text query with a facet on author
curl 'http://localhost:8983/solr/books/select?q=title_t:darkness&facet=true&facet.field=author_s'
```

The admin UI is at `http://localhost:8983/solr/`. For a guided tour, `bin/solr start -e techproducts` boots a preloaded example that exercises faceting, highlighting, and spellcheck.

## Architecture / How It Works

A Solr document is a flat set of fields; indexing runs each field through an analysis chain (tokenizer + filters) defined by its field type, producing terms in a Lucene inverted index. Queries are parsed by a pluggable **query parser** (`lucene`, `dismax`, `edismax` are the common ones), executed against that index, and post-processed by **search components** (faceting, highlighting, stats, spellcheck, MoreLikeThis). The unit of storage is a Lucene index directory; Solr calls a single index a **core**, and a logically-sharded, replicated index a **collection**.

**Schema.** Fields, field types, and analysis chains live in a schema. Solr supports a strict managed schema, a "schemaless" mode that infers field types from incoming data, and a `_default` configset you extend explicitly. Field types are where relevance is won or lost — analyzers, `docValues` (for sorting/faceting/function queries), and stored-vs-indexed decisions all live here.

**SolrCloud.** The distributed mode partitions a collection into **shards** and replicates each shard across nodes. Cluster state — which node hosts which replica, leader election, configset distribution — is coordinated through an external **ZooKeeper** ensemble. A document routes to a shard by a hash of its `id` (compositeId routing) unless you use implicit routing. There is an ongoing effort to offer coordination without a standalone ZooKeeper ensemble, but ZooKeeper remains the standard, supported topology.

**Commits.** Lucene segments are only searchable after a commit. Solr separates **hard commits** (fsync, durable, opens a new searcher optionally) from **soft commits** (make documents visible for near-real-time search without fsync). Getting commit cadence wrong is one of the most common causes of either poor indexing throughput or memory pressure from too many open searchers.

**Kubernetes.** The Solr Operator manages ZooKeeper, StatefulSets, and rolling updates on Kubernetes, and is the recommended way to run SolrCloud in a container orchestrator[^3].

## Production Notes

**Reindexing is the recurring tax.** Many schema changes — altering a field's analysis chain, adding `docValues`, changing a field type — do not apply to already-indexed documents. The supported answer is a full reindex, which for large corpora means maintaining a reindex pipeline and a re-source of truth outside Solr. Treat Solr as a derived index, never the system of record.

**Sharding is decided up front.** A collection's shard count is fixed at creation for compositeId routing. Growing beyond it means `SPLITSHARD` (online but I/O-heavy) or reindexing into a new collection behind an alias. Plan shard count against projected corpus size, not current size.

**JVM heap and the OS page cache.** Lucene memory-maps index files and relies on the operating system page cache for hot data, so a large JVM heap is usually the wrong lever — it steals RAM from the page cache and lengthens GC pauses. Common guidance is a moderate heap (often well under 32 GB to keep compressed oops) with the rest of RAM left free for the OS to cache the index. G1GC is the typical collector.

**ZooKeeper is a first-class operational dependency.** A SolrCloud cluster is only as healthy as its ZooKeeper ensemble: quorum loss makes the cluster read-only for state changes, and configset changes flow through ZooKeeper. It needs its own monitoring, its own disk, and its own upgrade discipline.

**Security is opt-in.** Out of the box, an exposed Solr instance is unauthenticated. Authentication, authorization, and TLS are configured through `security.json` and the admin API. Historically Solr has had serious CVEs around the `VelocityResponseWriter`, the DataImportHandler, and unsecured admin endpoints exposed to the internet — never bind an unsecured instance to a public interface.

**Upgrades cross a Lucene index-format boundary.** Solr generally supports reading indexes written by the immediately preceding major version, but not older; skipping a major version can require a full reindex. The 8.x → 9.x jump also moved the repository, changed the minimum Java version to 11, and dropped deprecated components, so it was a genuine migration rather than a drop-in.

## When to Use / When Not

**Use when:**
- You need rich faceted, filtered text search over structured documents (e-commerce, catalogs, libraries, content).
- You want a fully OSI-Apache-2.0 search server with no source-available license strings attached.
- You need fine-grained relevance control — custom analyzers, function queries, `edismax` boosting, reranking.
- You want hybrid lexical + dense-vector search from one engine without bolting on a separate vector store.

**Avoid when:**
- Your primary need is log/metrics/observability analytics — the Elasticsearch/OpenSearch ecosystem (Kibana/OpenSearch Dashboards, Beats, ingest tooling) is far better tooled for that.
- You want minimal operations — running SolrCloud plus a ZooKeeper ensemble is nontrivial next to a single-binary engine.
- You only need vector similarity search — a purpose-built vector database will have a simpler API and operational model.
- Your team has no JVM operations experience and no appetite to build one.

## Alternatives

- elastic/elasticsearch — use instead when you need log/observability analytics and the Kibana ecosystem; note it is SSPL/AGPL source-available, not OSI open source.
- opensearch-project/OpenSearch — use instead when you want the Elasticsearch feature set and tooling under a genuine Apache-2.0 license (the AWS-led fork).
- apache/lucene — use instead when you are building a JVM application and want to embed the search library directly, without running a separate server.
- meilisearch/meilisearch — use instead when you want instant-search with typo tolerance and a simple API, and don't need Solr's relevance depth or JVM operations.
- qdrant/qdrant — use instead when the workload is primarily dense-vector similarity search and lexical faceting is secondary.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.1 | 2007-01 | First Apache release after CNET donation[^1]. |
| 3.1 | 2011-03 | Version realigned with Lucene under the combined repo. |
| 4.0 | 2012-10 | SolrCloud — distributed indexing/search via ZooKeeper. |
| 5.0 | 2015-02 | Shipped as a standalone application, not a WAR. |
| 6.0 | 2016-04 | Streaming Expressions and Parallel SQL. |
| 7.0 | 2017-09 | Replica types, autoscaling framework. |
| 8.0 | 2019-03 | Last major line from the combined `lucene-solr` repo. |
| 9.0 | 2022-05 | First release from standalone `apache/solr`; Java 11 minimum, dense-vector KNN search[^2]. |

## References

[^1]: Apache Solr — project history and news. https://solr.apache.org/news.html
[^2]: Apache Solr Reference Guide, "Dense Vector Search". https://solr.apache.org/guide/solr/latest/query-guide/dense-vector-search.html
[^3]: Apache Solr Operator (Kubernetes) documentation. https://solr.apache.org/operator/

## Tags

java, search-engine, information-retrieval, full-text-search, vector-search, apache-lucene, solrcloud, faceted-search, jvm, distributed-systems
