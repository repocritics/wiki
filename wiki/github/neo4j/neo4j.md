# neo4j/neo4j

> The native property-graph database and the reference implementation of the Cypher query language.

[GitHub repo](https://github.com/neo4j/neo4j) ·
[Official website](https://neo4j.com) ·
[License: GPL-3.0](https://github.com/neo4j/neo4j/blob/2026.06/LICENSE.txt)

## Overview

Neo4j is a database that stores data as a property graph — nodes and typed, directed relationships, each carrying arbitrary key/value properties — rather than as tables or documents[^1]. First released in 2010, it is the most widely deployed graph database and the origin of Cypher, the declarative graph query language that later became the basis of the ISO GQL standard[^2]. Typical uses are fraud detection, recommendation engines, network/IT topology, identity and access graphs, knowledge graphs, and increasingly retrieval-augmented generation over linked data.

The defining property is **index-free adjacency**: each node holds direct pointers to its relationships, so traversing from a node to its neighbors is a pointer chase rather than an index lookup or join. This makes deep, variable-length traversals (friend-of-friend, shortest path, k-hop) roughly constant per hop regardless of total database size — the workload where relational recursive joins degrade. The tradeoff is that Neo4j is not a general-purpose store: aggregations over the whole dataset, wide analytical scans, and high-write-throughput ingestion are weaker than in columnar or relational systems.

The most important thing to understand before adopting Neo4j is the **Community/Enterprise split**. This repository is Neo4j Community Edition, GPLv3[^3]. The features most people assume "Neo4j has" — clustering and high availability, online/hot backup, fine-grained role-based access control, multiple active databases beyond the defaults, and query-level observability — are in the closed-source Enterprise Edition, which requires a commercial license or the managed Aura service[^4]. Evaluating on Community and deploying on Enterprise is a common and expensive surprise.

## Getting Started

The fastest path is the official Docker image (building from source needs Maven 3.8.2+, JDK 17, and 2 GB+ of Maven heap[^5]):

```bash
docker run --rm \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password123 \
  neo4j:community
```

Open the browser UI at `http://localhost:7474`, or connect over the Bolt protocol on `7687`. A minimal Cypher session:

```cypher
// Create nodes and a relationship
CREATE (tom:Person {name: 'Tom'})
CREATE (brad:Person {name: 'Brad'})
CREATE (tom)-[:KNOWS {since: 2024}]->(brad);

// Index the property you match on — otherwise every MATCH scans all :Person
CREATE INDEX person_name IF NOT EXISTS FOR (p:Person) ON (p.name);

// Traverse: who does Tom know, up to 2 hops out?
MATCH (t:Person {name: 'Tom'})-[:KNOWS*1..2]->(other)
RETURN DISTINCT other.name;
```

## Architecture / How It Works

Neo4j is a JVM application (Java, with some native components). The engine has a few load-bearing layers:

- **Native store files.** Nodes, relationships, properties, and labels are stored in fixed-size record files. A node record points to its first relationship and first property; relationships form doubly linked lists per node. This layout is what makes index-free adjacency possible — but it also means a node with millions of relationships (a "supernode" / dense node) has an expensive relationship chain to walk.
- **Page cache.** The store files are memory-mapped through Neo4j's own page cache, separate from the JVM heap. For good traversal performance the working graph should fit in the page cache; a graph that spills to disk on every hop loses the constant-per-hop property.
- **Cypher planner and runtime.** Cypher is compiled to a logical then physical plan. The cost-based planner uses store statistics and indexes to pick access paths; `EXPLAIN` shows the plan, `PROFILE` shows real row/db-hit counts. Enterprise ships a faster pipelined runtime; Community uses the slotted/interpreted runtime.
- **Bolt protocol.** A binary, statement-oriented client protocol introduced in 3.0, spoken by the official drivers (Java, Python, JavaScript, Go, .NET). HTTP APIs exist but Bolt is the production path.
- **Transactions.** Full ACID with write-ahead logging and a single write lock model per entity. Reads see a consistent snapshot.

Two ecosystem libraries are close enough to be treated as part of the platform: **APOC** (a large standard-library of procedures — data import, graph refactoring, utilities) and **GDS** (Graph Data Science — PageRank, community detection, node embeddings, pathfinding at scale)[^6]. Neither is in this repo; both are installed as plugins and version-locked to the server.

## Production Notes

**Memory tuning is the operator's main job.** Two independent pools must be sized: JVM heap (for query execution and transaction state) and page cache (for store files). Under-sizing the page cache is the classic cause of "graph got slow as it grew." `neo4j-admin server memory-recommendation` gives a starting point; leave headroom for the OS.

**Dense nodes are a modeling footgun, not a config one.** A node with millions of relationships makes any traversal through it slow and creates lock contention on writes touching it. The fix is a data-model change (relationship-type partitioning, intermediate nodes), not a flag.

**Write scaling is bounded by a single leader.** In an Enterprise cluster, writes go to one Raft leader and replicate to followers; read replicas scale reads, not writes[^4]. There is no built-in horizontal write sharding of a single database — if you need that, Neo4j is the wrong shape.

**Backups differ by edition.** Community has only offline `neo4j-admin database dump/load` (stop the DB, or accept an inconsistent copy). Online/consistent hot backup is Enterprise-only. Plan your DR story around the edition you will actually run.

**Bulk load is separate from Cypher.** For initial millions-to-billions of records, use `neo4j-admin database import` (offline, CSV) — orders of magnitude faster than `CREATE`/`MERGE`. Incremental loads via `LOAD CSV` should batch with `CALL { ... } IN TRANSACTIONS` to avoid one giant transaction blowing the heap.

**Query footguns.** Missing indexes turn `MATCH` into full label scans; unintended cartesian products appear when disconnected patterns share no variable; `MERGE` without a supporting uniqueness constraint can create duplicates under concurrency. Always `PROFILE` before blaming the engine.

**Upgrades are store-format events.** Major jumps (3.x → 4.x → 5.x, and the move to calendar-versioned releases in 2025) have involved store-format migrations, config renames, and Cypher deprecations. Read the migration guide, test on a copy, and budget downtime; rolling upgrades are an Enterprise-cluster feature.

## When to Use / When Not

**Use when:**
- Relationships are first-class in your queries — deep traversals, pathfinding, pattern matching on connections.
- Your schema is evolving or heterogeneous and rigid relational modeling hurts.
- You want a mature Cypher/GQL query language and a large driver/tooling ecosystem.
- Graph algorithms (centrality, community, similarity, embeddings) are part of the workload.

**Avoid when:**
- Your access pattern is key/value, wide analytical scans, or high-volume append ingestion — a relational, columnar, or document store fits better.
- You need horizontal write sharding of one logical graph across many nodes.
- You need HA, hot backup, or RBAC but cannot pay for Enterprise or Aura — Community alone will not cover production HA.
- The data is fundamentally tabular and relationships are rare; SQL with occasional recursive CTEs is simpler and cheaper.

## Alternatives

- memgraph/memgraph — in-memory, Cypher-compatible; use instead when you need low-latency, streaming/real-time graph analytics.
- janusgraph/janusgraph — distributed over Cassandra/HBase, Gremlin; use instead when you need horizontal write scale beyond a single leader.
- dgraph-io/dgraph — distributed, GraphQL-native; use instead when you want built-in sharding and a GraphQL API.
- arangodb/arangodb — multi-model (document + graph + key/value); use instead when your workload mixes documents and graphs.
- apache/age — a Postgres extension adding openCypher; use instead when you want graph queries inside an existing PostgreSQL deployment.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010-02 | First stable release; embedded Java graph store. |
| 2.0 | 2013-12 | Node labels; Cypher promoted to primary query language. |
| 3.0 | 2016-04 | Bolt binary protocol; user-defined stored procedures. |
| 4.0 | 2020-01 | Multiple databases; reactive drivers; fine-grained RBAC (Enterprise). |
| 5.0 | 2022-10 | Autonomous clustering; new index engine; store-format changes[^1]. |
| 2025.01 | 2025 | Shift to calendar-versioned release train[^2]. |
| 2026.06 | 2026-06 | Current release line (repo default branch). |

## References

[^1]: Neo4j documentation — "Introduction" and property graph model. https://neo4j.com/docs/getting-started/
[^2]: ISO/IEC 39075 GQL standard (2024); Cypher and openCypher lineage. https://opencypher.org/
[^3]: Neo4j Community Edition license (GPLv3), this repository. https://github.com/neo4j/neo4j/blob/2026.06/LICENSE.txt
[^4]: Neo4j clustering and Enterprise features overview. https://neo4j.com/docs/operations-manual/current/clustering/
[^5]: Neo4j build instructions (Maven 3.8.2, JDK 17). https://github.com/neo4j/neo4j/blob/2026.06/README.asciidoc
[^6]: Neo4j Graph Data Science library. https://neo4j.com/docs/graph-data-science/current/

## Tags

java, graph-database, cypher, nosql, gql, property-graph, database, bolt-protocol, knowledge-graph, jvm, acid
