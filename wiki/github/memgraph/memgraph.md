# memgraph/memgraph

> In-memory, Cypher-compatible graph database in C++, positioned as the graph engine for GraphRAG and real-time analytics.

[GitHub repo](https://github.com/memgraph/memgraph) ·
[Official website](https://memgraph.com) ·
[License: BSL 1.1 / MEL — source-available, not OSI](https://github.com/memgraph/memgraph/blob/master/licenses/BSL.txt)

## Overview

Memgraph is an in-memory graph database written in C++ that speaks the same Cypher query language and Bolt wire protocol as Neo4j, so most Neo4j drivers connect to it unchanged[^1]. The company (Memgraph Ltd., founded 2016) opened the core engine on GitHub in 2020–2021 and has since repositioned the product around AI workloads: GraphRAG retrieval, agent memory, and real-time graph analytics for fraud, network, and infrastructure use cases[^2]. The current pitch is "structured, connected context alongside vector and text search in one query layer" — built-in vector/text indexes plus graph traversal in a single Cypher statement.

The defining tradeoff is right in the name: Memgraph keeps the working graph **in RAM**. This is what buys sub-millisecond multi-hop traversals, and it is also the hardest operational constraint — your graph (plus index overhead) has to fit in memory, and durability rests on periodic snapshots and a write-ahead log rather than a disk-resident store. Neo4j, the obvious comparison, is disk-first with a page cache; Memgraph inverts that. Later on-disk and analytical storage modes soften the limit but change the guarantees.

The second thing to be honest about is licensing. Despite "open-source" in the repository description, Memgraph Community is under the **Business Source License 1.1** — source-available, with production-use restrictions that convert to Apache 2.0 on a time delay — and Memgraph Enterprise (HA, RBAC, SSO, multi-tenancy) is under the proprietary MEL[^3]. GitHub reports the license as `NOASSERTION`. This is a real distinction from Neo4j Community (GPLv3) and from genuinely OSI-licensed graph stores.

## Getting Started

The supported path is Docker; there is no Homebrew/apt one-liner for the engine itself.

```bash
docker run -p 7687:7687 -p 7444:7444 memgraph/memgraph-mage
# 7687 = Bolt (query), 7444 = monitoring; memgraph-mage bundles the algorithm library
```

Connect with any Neo4j-compatible driver — here the Python `neo4j` package:

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("", ""))
with driver.session() as s:
    s.run("CREATE (:Person {name: $n})-[:KNOWS]->(:Person {name: $m})",
          n="Tom", m="Brad")
    rows = s.run(
        "MATCH (p:Person)-[:KNOWS]->(f) RETURN p.name AS a, f.name AS b"
    )
    for r in rows:
        print(r["a"], "->", r["b"])
driver.close()
```

For interactive use, Memgraph Lab (a separate desktop/web UI) and the `mgconsole` CLI are the first-party clients.

## Architecture / How It Works

**Storage modes are the core design axis**, and picking the wrong one is the most common footgun:

- `IN_MEMORY_TRANSACTIONAL` (default) — full ACID, MVCC snapshot isolation, WAL + periodic snapshots for durability. The mode you run in production.
- `IN_MEMORY_ANALYTICAL` — drops transactional isolation and most locking to make bulk import and read-heavy analytics much faster, at the cost of ACID guarantees and safe concurrent writes. Used to load large datasets fast, then optionally switch back.
- `ON_DISK_TRANSACTIONAL` — a disk-resident store for graphs larger than RAM, trading the in-memory latency profile for capacity.

**Query execution.** Cypher is parsed and planned into an operator tree executed by the C++ engine. Memgraph supports parallel query execution and deep-path constructs (weighted shortest path, breadth/depth-first expansion, path filtering with accumulators) as query primitives rather than application-side loops.

**MAGE** (Memgraph Advanced Graph Extensions) is the algorithm/procedure library — the counterpart to Neo4j's GDS and APOC — shipping PageRank, community detection, link prediction, embeddings, and dynamic (streaming) variants, implemented in C++, Python, and CUDA. Custom query modules can be written in Python, C/C++, or Rust and called from Cypher. The `memgraph/memgraph-mage` image bundles it; the plain `memgraph/memgraph` image does not.

**Streaming and ingestion.** Native consumers for Kafka, Pulsar, and Redpanda let the graph mutate from a stream, and MAGE's dynamic algorithms recompute incrementally. Bulk loading supports CSV, JSON, Parquet, and JSONL from local disk, S3, or HTTP.

**High availability** (Enterprise) is a Raft-coordinated single-main / multi-replica topology with automatic failover — writes go to the main, replicas serve reads. This is replication for availability, not a sharded multi-writer cluster; a single graph still lives on a single machine's memory.

## Production Notes

- **Capacity planning is memory planning.** Size RAM for the graph *plus* index structures (label, property, vector, text) plus MVCC version overhead from long-running transactions. Under-provisioning surfaces as OOM kills, not graceful degradation. The analytical and on-disk modes exist precisely because the default in-memory transactional mode is RAM-bound.
- **Cypher-compatible is not drop-in-compatible with Neo4j.** The Bolt protocol and core Cypher match, so drivers work, but procedures differ (MAGE, not APOC/GDS), configuration and admin commands differ, and some Cypher clauses and semantics diverge. Migrations from Neo4j need query and tooling review, not just a connection-string swap.
- **Two images, easy to conflate.** `memgraph/memgraph` lacks MAGE; calling an algorithm procedure then fails at runtime. Use `memgraph/memgraph-mage` if you need algorithms.
- **Durability is snapshot + WAL.** Recovery replays the WAL and loads the last snapshot; recovery time and snapshot cadence are tunables that trade RSS/IO against potential data loss and restart latency. Test restart-from-cold with a production-sized dataset before you rely on it.
- **The enterprise cliff.** HA/failover, role- and label-based access control, SSO, multi-tenancy, and audit are Enterprise (MEL) features. A design that assumes automatic failover or fine-grained authorization is an Enterprise-licensed design, not a Community one.
- **Issue backlog.** The repository carries a large open-issue count (700+), consistent with an actively developed C++ database rather than an abandoned one; last pushes are current. Read it as active velocity plus a broad surface area, not as instability by itself.

## When to Use / When Not

**Use when:**
- You need low-latency multi-hop traversals on a graph that fits in memory (real-time recommendations, fraud rings, network/dependency analysis).
- You're building GraphRAG or agent memory and want graph traversal, vector search, and text search co-located in one Cypher query.
- You already know Cypher / the Neo4j driver ecosystem and want an in-memory engine with streaming ingestion.

**Avoid when:**
- Your graph is far larger than affordable RAM and latency isn't the priority — a disk-first store (Neo4j) or the on-disk mode's different profile may fit better.
- You require an OSI-approved open-source license with no production-use restrictions — BSL/MEL will not satisfy that policy.
- Your workload is relational or document-shaped without genuine graph traversal — a graph database is overhead you won't recoup.
- You need horizontal write sharding across machines; the model is single-main replication, not a distributed multi-writer cluster.

## Alternatives

- neo4j/neo4j — the disk-first incumbent; use it when the graph exceeds RAM, you want the largest ecosystem (APOC/GDS), or you need a GPLv3/commercial license story.
- apache/age — Postgres extension adding openCypher; use it when you want graph queries inside an existing PostgreSQL deployment rather than a separate engine.
- arangodb/arangodb — multi-model (graph + document + key-value); use it when you want graph plus document storage in one system.
- JanusGraph/janusgraph — distributed, Gremlin-based, backed by Cassandra/HBase/Bigtable; use it when horizontal scale over huge graphs matters more than in-memory latency.
- qdrant/qdrant — a dedicated vector database; use it when your retrieval need is similarity search without real graph traversal.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016 | Memgraph Ltd. founded; engine developed as a commercial product. |
| Repo public | 2020-09 | Source published on GitHub (repository created 2020-09-21). |
| Core relicensed | 2021 | Community engine moved to source-available BSL; MAGE library released. |
| 2.x | 2022–2023 | Analytical storage mode, expanded MAGE, streaming consumers, vector/text indexing added over the line. |
| 3.x | 2024–2026 | GraphRAG / AI-memory positioning, AI Toolkit + MCP server, hybrid vector-plus-graph retrieval. |

## References

[^1]: Memgraph README, "Description" — Cypher compatibility with Neo4j, Bolt protocol, C/C++ in-memory engine. https://github.com/memgraph/memgraph
[^2]: Memgraph documentation and product site. https://memgraph.com/docs
[^3]: Memgraph licensing — Community under the Business Source License (BSL 1.1), Enterprise under the Memgraph Enterprise License (MEL). https://github.com/memgraph/memgraph/blob/master/licenses/BSL.txt

## Tags

graph-database, cypher, in-memory, cpp, graphrag, ai-memory, streaming, vector-search, neo4j-compatible, real-time-analytics, source-available, database
