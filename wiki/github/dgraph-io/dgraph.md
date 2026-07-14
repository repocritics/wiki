# dgraph-io/dgraph

> Distributed, sharded graph database in Go — RDF/GraphQL data model, Raft-replicated, predicate-sharded for horizontal scale.

[GitHub repo](https://github.com/dgraph-io/dgraph) ·
[Official website](https://dgraph.io) ·
[License: Apache-2.0](https://github.com/dgraph-io/dgraph/blob/main/LICENSE)

## Overview

Dgraph is a horizontally scalable graph database written in Go. It stores data as RDF-style triples (subject–predicate–object), replicates each shard with Raft, and shards the dataset **by predicate** rather than by node — the design decision that shapes almost everything about how it scales and where it breaks[^1]. It exposes two query surfaces over gRPC and HTTP: DQL (Dgraph Query Language, formerly "GraphQL+-"), and a native GraphQL layer that auto-generates a CRUD API from a schema you supply[^2].

The intended user is someone with genuinely graph-shaped data at scale — many-to-many relationships, deep traversals, sparse attributes that fit poorly in SQL tables — who also wants distributed ACID transactions and consistent replication rather than the single-server ceiling of most property-graph engines. Dgraph's own framing is "Google-scale throughput with low-latency real-time queries," and its comparison table positions it against Neo4j (single-server in the community edition) and JanusGraph (a layer over other stores)[^1].

The defining tension is the predicate sharding model. It gives Dgraph automatic rebalancing and clean horizontal scale for datasets with many predicates — but a single predicate is the unit of distribution and cannot be split across groups, so one very high-cardinality relationship becomes a hotspot no amount of nodes will relieve. The second tension is organizational: Dgraph Labs hit financial trouble and layoffs in 2021, development slowed to a near-standstill for roughly two years, and stewardship later moved to Hypermode (the company co-founded by Dgraph's original author)[^3]. The v25 line and the `main` branch are the active continuation of that history.

## Getting Started

Dgraph officially supports only Linux/amd64 and Linux/arm64; official Mac and Windows binaries were dropped in 2021, and Docker is the recommended path[^4].

```bash
# Single-node cluster (Zero + Alpha bundled) for local development
docker run -it -p 8080:8080 -p 9080:9080 \
  -v ~/dgraph:/dgraph dgraph/standalone:latest
```

Set a schema and run a query with DQL:

```bash
# Define a predicate with an index
curl -X POST localhost:8080/alter -d '
  name: string @index(term) .
'

# Mutate (RDF n-quads)
curl -X POST localhost:8080/mutate?commitNow=true \
  -H 'Content-Type: application/rdf' -d '{
    set { _:tom <name> "Tom" . }
  }'

# Query
curl -X POST localhost:8080/query -H 'Content-Type: application/dql' -d '
  { people(func: has(name)) { uid name } }'
```

## Architecture / How It Works

Dgraph runs as two distinct binaries, both members of Raft groups:

- **Dgraph Zero** — the control plane. It hands out UID and transaction-timestamp leases, tracks group membership, and orchestrates data movement when it rebalances predicates across groups. Zero itself is a Raft group (run 3 or 5 for HA); losing Zero quorum halts new transactions[^5].
- **Dgraph Alpha** — the data plane. Each Alpha group owns a set of predicates and their indexes and serves queries and mutations. Alphas within a group replicate via Raft; groups are independent Raft clusters.

Data is stored as **posting lists**: for each predicate, the list of (subject → object) entries, persisted in Badger, Dgraph's own embedded LSM-tree key-value store[^6]. Predicate-level locality is deliberate — a query that touches one predicate hits one group, minimizing cross-network fan-out. It also means the predicate is the atomic shard: Zero can move a predicate between groups to rebalance, but it cannot split a single predicate.

Transactions are **MVCC and lock-free**, using monotonic timestamps issued by Zero; Dgraph advertises linearizable reads and distributed ACID commits. A query that spans predicates in multiple groups performs distributed joins and traversals across those groups, coordinated by the Alpha that received the request.

The GraphQL layer sits on top of DQL: you upload a GraphQL SDL schema, and Dgraph generates queries, mutations, and the underlying DQL/index plumbing. DQL remains the lower-level surface with access to Dgraph-specific traversal features (variables, shortest-path, recurse, facets) that the generated GraphQL API does not expose.

## Production Notes

- **The predicate hotspot is the real ceiling.** Because a predicate cannot be split across groups, a single dominant edge (e.g. one relationship every node participates in) concentrates load on one group regardless of cluster size. Model schemas to avoid one giant predicate; this is the failure mode operators hit at scale, not raw node count.
- **Memory-hungry.** Badger plus in-memory posting-list caches make Dgraph RAM-sensitive; undersized Alphas OOM under query load. Size for working-set-in-memory, not just disk.
- **Zero is a hard dependency.** No timestamps, no UID leases, no rebalancing without a healthy Zero quorum. Treat it as critical infrastructure alongside the Alphas, not an afterthought.
- **Enterprise-gated features.** ACLs, encryption at rest, binary backups, and learner/read-replica nodes live behind an enterprise license; the open-source build gives you `export` (logical) but not incremental binary backups. Plan backup/restore around what your license tier actually includes.
- **Upgrade friction across major versions.** Storage and internal formats have changed across the big version jumps, and the recommended path between some majors is export-and-reimport rather than in-place upgrade. Budget migration windows; read the release notes for the specific hop.
- **The maintenance gap left scars.** Roughly 2021–2023 saw minimal releases and community uncertainty (some users forked or migrated). The project is active again under Hypermode, but evaluate the current release cadence yourself rather than assuming either the old momentum or the old stall[^3].
- **Linux-only in production.** Non-Linux builds are unsupported for serving; don't plan a Windows/Mac production deployment[^4].

## When to Use / When Not

**Use when:**
- Your data is genuinely graph-shaped (deep traversals, many-to-many, sparse attributes) and larger than a single server can hold.
- You need distributed ACID transactions and consistent replication, not just a single-node graph engine.
- You want a native GraphQL API generated from a schema, or Dgraph-specific DQL traversals.
- Your access patterns spread across many predicates rather than hammering one.

**Avoid when:**
- Your workload centers on one enormous predicate — the sharding model will bottleneck it.
- You have fewer than ~10 related tables and SQL joins already suffice; a relational DB is simpler.
- You need Cypher/openCypher compatibility or a mature property-graph tooling ecosystem — that's Neo4j's turf.
- You need Windows/Mac production support, or you require the enterprise-only features (ACL/encryption/binary backup) but can't take the license.

## Alternatives

- neo4j/neo4j — mature property-graph database with Cypher; use it when you want the richest graph tooling and single-server (or enterprise-clustered) is enough.
- janusgraph/janusgraph — distributed graph layered over Cassandra/HBase with Gremlin; use it when you already run those stores and want Apache TinkerPop compatibility.
- arangodb/arangodb — multi-model (document + graph + key-value); use it when your data isn't purely graph and you want one engine for several shapes.
- surrealdb/surrealdb — newer multi-model DB in Rust with a SQL-like language and graph edges; use it when you want document+graph+auth in a single modern server.
- memgraph/memgraph — in-memory, Cypher-compatible graph engine; use it when latency on a working set that fits in RAM matters more than horizontal sharding.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017-12 | First production-ready GA. |
| 1.1 | 2019-09 | Type system, `@upsert` improvements. |
| 20.03 | 2020-03 | Calendar versioning; native GraphQL API and Slash GraphQL managed service. |
| 21.03 | 2021-04 | Last release before the extended maintenance gap; Mac/Windows support dropped[^4]. |
| 23.0 | 2023 | Development resumes under Hypermode stewardship[^3]. |
| 24.0 | 2024 | Continued releases on the revived line. |
| 25.0 | 2025 | Current production line (`main`)[^7]. |

## References

[^1]: Dgraph README and feature comparison — architecture, sharding, and DB comparison table. https://github.com/dgraph-io/dgraph
[^2]: Dgraph GraphQL and DQL documentation. https://docs.dgraph.io/graphql
[^3]: Hypermode — company continuing Dgraph's development after Dgraph Labs' 2021 downturn. https://hypermode.com
[^4]: "Dropping support for Windows and Mac" — Dgraph discuss, 2021. https://discuss.dgraph.io/t/dropping-support-for-windows-and-mac/12913
[^5]: Dgraph design concepts — Zero, Alpha, groups, and Raft. https://docs.dgraph.io/design-concepts/
[^6]: BadgerDB — the embeddable LSM key-value store Dgraph stores posting lists in. https://github.com/dgraph-io/badger
[^7]: Dgraph v25.0.0 release. https://github.com/dgraph-io/dgraph/releases/tag/v25.0.0

## Tags

graph-database, distributed-database, go, graphql, rdf, raft, database, knowledge-graph, acid, sharding
