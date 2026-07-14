# arangodb/arangodb

> A native multi-model database — documents, graphs, and key-value in one engine, queried by a single SQL-like language (AQL).

[GitHub repo](https://github.com/arangodb/arangodb) ·
[Official website](https://www.arangodb.com) ·
[License: BUSL-1.1](https://github.com/arangodb/arangodb/blob/devel/LICENSE)

## Overview

ArangoDB is a database written in C++ that stores three data models in one engine: JSON documents, property graphs, and key-value pairs. The selling point is that a single query language, AQL, spans all three — you can filter documents, traverse a graph, and run a full-text search in one query without stitching together separate systems[^1]. It started life as "AvocadoDB" in 2011 (the avocado still appears in the logo), was renamed ArangoDB in 2012, and is developed commercially by ArangoDB GmbH / ArangoDB, Inc.[^2]

The defining tension is **multi-model generalism versus specialist depth**. ArangoDB does documents about as well as MongoDB-class stores and graphs about as well as a mid-tier graph database, but a team that only needs one model can usually find a specialist (Neo4j for graphs, MongoDB/Postgres for documents) that is deeper on tooling, driver maturity, and hosting options. ArangoDB earns its place when a workload genuinely mixes models — the classic example being a graph where every vertex and edge is a rich JSON document, traversed and filtered in the same query.

The second, larger tension as of 2026 is **licensing**. ArangoDB was Apache 2.0 for most of its life. With the 3.12 line (Licensed Work release date March 2024) the self-managed edition moved to the Business Source License 1.1 (BUSL-1.1): source-available, free for internal production use, but restricted from being offered as a competing commercial/hosted service until each release's four-year change date, at which point that release reverts to Apache 2.0[^3]. This is not open source by the OSI definition, and it reshaped who can build on ArangoDB.

## Getting Started

```bash
# Fastest path: Docker (Community Edition)
docker run -d -e ARANGO_ROOT_PASSWORD=test123 -p 8529:8529 arangodb
# Web UI + HTTP API on http://127.0.0.1:8529
```

AQL over a mixed document/graph dataset:

```aql
// Documents that are also graph vertices, filtered then traversed
FOR user IN users
  FILTER user.active == true
  FOR friend, edge IN 1..2 OUTBOUND user knows
    FILTER friend.city == "Seoul"
    RETURN DISTINCT { user: user.name, friend: friend.name, hops: LENGTH(edge) }
```

Collections are schema-free by default; you opt into JSON-schema validation per collection. Edges live in "edge collections" (documents with reserved `_from` / `_to` fields), and a named graph binds vertex and edge collections together for traversal.

## Architecture / How It Works

**Storage engine.** Since 3.4 the sole storage engine is **RocksDB** (an LSM-tree key-value store)[^4]. The older memory-mapped "MMFiles" engine was deprecated and removed in 3.7, so all data — documents, edges, indexes — is ultimately RocksDB key ranges. This gives durable, larger-than-RAM datasets and document-level locking, but inherits LSM characteristics: write amplification, compaction pauses, and the fact that "count" and range scans are not always O(1).

**Query execution.** AQL is parsed into an execution plan of pipelined nodes, run through a cost-based optimizer with pluggable rules, and (in a cluster) split into per-DBServer fragments coordinated centrally. AQL is expressive — subqueries, joins, graph traversals, aggregation, and geo/full-text all in one grammar — but it is its own language, not SQL, so existing SQL tooling and ORMs do not apply.

**Graph traversal** is a first-class AQL construct (`FOR v, e, p IN min..max OUTBOUND/INBOUND/ANY`). On a single server this is efficient; across a sharded cluster, traversals that cross shard boundaries incur network hops per hop-level unless the graph is laid out to keep connected data co-located (see SmartGraphs below).

**ArangoSearch** is an integrated full-text and ranking engine built on the IResearch library, exposed as "views" and BM25/TF-IDF scoring inside AQL[^5]. It removes the need for a separate Elasticsearch for many use cases, at the cost of being less feature-rich than a dedicated search stack.

**Cluster topology** has three roles: **Agents** (a Raft-based consensus group, the "Agency," holding cluster configuration and health), **Coordinators** (stateless AQL/HTTP entry points that plan and fan out queries), and **DBServers** (shard owners holding the actual RocksDB data)[^6]. Sharding is by `_key` hash by default. Automatic failover promotes a follower shard replica when a DBServer dies.

**Foxx** embeds a JavaScript runtime inside the server so you can ship data-adjacent microservices that run next to the data. It is a genuine differentiator but also a maintenance surface; newer releases have been reducing the footprint of the embedded JavaScript engine, so treat heavy Foxx investment as a bet that needs version-by-version verification.

## Production Notes

**The Community Edition has a dataset size limit.** Alongside the license change, the free Community Edition gained a per-deployment dataset size cap (documented around 100 GB); lifting it requires the Enterprise Edition[^7]. This is the single biggest "read the fine print" item — a proof-of-concept that fits comfortably can hit a wall in production without a code change, only a licensing one.

**Enterprise-gated performance features.** The scaling features most often needed at cluster scale — **SmartGraphs** and **EnterpriseGraphs** (co-locating connected graph data to avoid cross-shard traversal hops), **SmartJoins** (co-located joins), **OneShard** (pinning a whole database to one DBServer to get single-server ACID semantics with cluster resilience), **Hot Backups**, encryption-at-rest, and auditing — are Enterprise-only. On the Community Edition, large-graph traversals in a sharded cluster can be dominated by network round-trips.

**Cluster operations are non-trivial.** A three-role cluster (Agents/Coordinators/DBServers) plus replication is more moving parts than a single-node document store. The Kubernetes operator (`kube-arangodb`) is the supported path and is reasonable, but capacity planning, resharding, and understanding shard/replica placement are real operational work. Many teams run OneShard or single-server precisely to avoid distributed-traversal complexity.

**RocksDB tuning matters at scale.** Write-heavy workloads expose compaction and write-stall behavior; memory is split between RocksDB block cache, the AQL/edge caches, and V8/Foxx. Undersized block cache on large working sets produces latency cliffs. Monitor compaction backlog and cache hit rates, not just CPU.

**Upgrade posture.** Upgrades within the 3.x line are generally in-place with a documented compatibility window, but the license boundary means new releases carry the BUSL terms and a fresh four-year clock. Organizations that require an OSI-approved license must either stay on the last Apache-2.0 release (3.11, which ages out of security support) or wait for each 3.12+ release to reach its change date.

## When to Use / When Not

**Use when:**
- Your workload genuinely mixes models — a graph of rich JSON documents you traverse *and* filter *and* full-text search in one query.
- You want one operational system instead of a graph DB + document DB + search cluster.
- You need graph traversal with document-shaped vertices and a single query language over both.
- BUSL-1.1 is acceptable and your dataset fits the Community cap or you'll buy Enterprise.

**Avoid when:**
- You only need one model — a specialist (Neo4j, MongoDB, Postgres, Redis) will have deeper tooling and a larger hiring pool.
- You require an OSI-approved open-source license or plan to offer a hosted service built on it.
- Your Community-Edition production dataset will exceed the size cap and Enterprise isn't budgeted.
- Your graph is huge and sharded and you can't pay for SmartGraphs to avoid cross-shard traversal costs.

## Alternatives

- neo4j/neo4j — dedicated property-graph database with Cypher; use instead when graph is your only model and you want the deepest graph tooling and ecosystem (also source-available/GPL, not permissive).
- mongodb/mongo — document store; use instead when you only need documents and want the broadest managed-hosting and driver options.
- surrealdb/surrealdb — newer multi-model (document + graph + more); use instead when you want a modern multi-model engine and can accept a young ecosystem (BSL-licensed too).
- dgraph-io/dgraph — distributed, GraphQL-native graph database; use instead when you want horizontally-scaled graph-first storage with a GraphQL interface.
- redis/redis — in-memory key-value; use instead when you only need KV/caching and latency is the dominant requirement.

OrientDB (orientechnologies/orientdb) was the closest historical multi-model peer but is effectively unmaintained; treat it as a cautionary comparison, not a live option.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011 | Started as "AvocadoDB"[^2]. |
| 1.0 | 2012 | Renamed ArangoDB; first stable line. |
| 2.0 | 2014 | Graph features and AQL maturation. |
| 3.0 | 2016 | VelocyPack internal format, cluster rework. |
| 3.2 | 2017 | RocksDB storage engine introduced (alongside MMFiles)[^4]. |
| 3.4 | 2018 | RocksDB becomes default; ArangoSearch added[^5]. |
| 3.7 | 2020 | MMFiles engine removed — RocksDB only. |
| 3.11 | 2023 | Last Apache-2.0 line. |
| 3.12 | 2024 | License changed to BUSL-1.1; Community dataset size cap[^3]. |

## References

[^1]: ArangoDB README and feature overview. https://github.com/arangodb/arangodb
[^2]: ArangoDB company / project history (originally AvocadoDB, renamed 2012). https://www.arangodb.com/company/
[^3]: ArangoDB LICENSE (Business Source License 1.1, Licensed Work release date March 2024, change license Apache 2.0). https://github.com/arangodb/arangodb/blob/devel/LICENSE
[^4]: ArangoDB storage engine (RocksDB) documentation. https://docs.arangodb.com/stable/components/arangodb-server/storage-engine/
[^5]: ArangoSearch (IResearch-based full-text and ranking). https://docs.arangodb.com/stable/index-and-search/arangosearch/
[^6]: ArangoDB cluster architecture (Agents/Coordinators/DBServers). https://docs.arangodb.com/stable/deploy/cluster/
[^7]: ArangoDB Community vs Enterprise Edition (dataset size limit). https://arangodb.com/community-server/

## Tags

cpp, database, multi-model, graph-database, document-database, key-value, nosql, distributed-database, aql, full-text-search, busl, source-available
