# surrealdb/surrealdb

> A multi-model (document + graph + relational) database written in Rust, with a SQL-like query language, built-in realtime queries, and row-level auth — shipped as a single binary under a source-available license.

[GitHub repo](https://github.com/surrealdb/surrealdb) ·
[Official website](https://surrealdb.com) ·
[License: BSL-1.1](https://github.com/surrealdb/license) (source-available, not OSI-approved)

## Overview

SurrealDB is a database that tries to collapse several data models — document, graph, relational, time-series, geospatial, and key-value — into one engine queried through a single SQL-like language called SurrealQL[^1]. It is built in Rust and distributed as one binary that can run in-memory, embedded inside an application (including in the browser via WebAssembly), as a single self-hosted node, or as a distributed cluster. The pitch is to fold the API/backend layer into the database itself: with built-in authentication, row-level permissions, and realtime "live queries," it markets itself as a backend-as-a-service, not just a store.

The project was founded by Tobie and Jaime Morgan Hitchcock (SurrealDB Ltd, London). The repository opened in December 2021, spent a long stretch in `1.0.0-beta` releases, and reached a `1.0` general-availability milestone in September 2023[^2]. It has grown quickly in visibility (~33k stars) but remains young for a database — the class of software where maturity is measured in decade-old durability track records, not stars.

The defining tension is **breadth versus maturity, and openness versus licensing**. SurrealDB is not OSI open source: it ships under the Business Source License 1.1, which forbids offering SurrealDB as a competing commercial database service; each version converts to Apache 2.0 several years after its release[^3]. GitHub reports the license as `NOASSERTION` because BSL is not on the SPDX standard-license list. Teams evaluating it for production must treat it as source-available, not free-software, infrastructure.

## Getting Started

```bash
# Install the single binary (macOS: brew install surrealdb/tap/surreal)
curl --proto '=https' --tlsv1.2 -sSf https://install.surrealdb.com | sh

# Start an in-memory dev server with root auth
surreal start --user root --pass root memory
```

```bash
# Connect an interactive SurrealQL shell in another terminal
surreal sql --endpoint http://localhost:8000 \
  --user root --pass root --ns test --db test --pretty
```

```surrealql
-- Schemafull table with a typed, asserted field and a unique index
DEFINE TABLE user SCHEMAFULL;
DEFINE FIELD email ON user TYPE string ASSERT string::is_email($value);
DEFINE INDEX email ON user COLUMNS email UNIQUE;

CREATE user SET email = "tobie@example.com";

-- Directed graph edge, then a graph traversal from a record id
RELATE user:tobie->wrote->article:surreal SET at = time::now();
SELECT ->wrote->article.* FROM user:tobie;
```

For persistence, replace `memory` with a storage backend such as `rocksdb://mydata.db` or `surrealkv://mydata.db`. SDKs exist for Rust, JavaScript/TypeScript, Python, Go, .NET, PHP, and Java, plus HTTP and WebSocket RPC.

## Architecture / How It Works

SurrealDB separates a **compute layer** (SurrealQL parser, query planner, permissions, functions) from a pluggable **key-value storage layer**. Every record, index entry, and edge is encoded as keys in the underlying KV store; transactions are built on the KV engine's own transaction support. Choosing a backend is the main deployment decision:

- **Embedded / single-node** — in-memory, RocksDB, or **SurrealKV** (SurrealDB's own Rust LSM-tree engine). These run inside one process; SurrealKV is the newer native persistent option and was beta for a long time.
- **Distributed** — TiKV and FoundationDB back a horizontally scalable cluster. This is where the "scale to hundreds of nodes" claim lives, and where operational cost rises sharply.

**SurrealQL** is the surface everyone touches. It reads like SQL but adds record links (`user:tobie`), graph edges via `RELATE` and arrow traversals (`->wrote->article`), nested/embedded objects, computed fields, table events, and typecasting. There is no `JOIN`; relationships are expressed through record links and graph edges instead. A GraphQL interface and an MCP (Model Context Protocol) server for LLM/agent access are also provided.

**Realtime** is a first-class feature: `LIVE SELECT` registers a query whose results stream to the client over WebSocket as underlying data changes. Combined with `DEFINE ACCESS` / record-level `PERMISSIONS`, this is what lets clients talk to the database directly — the BaaS story. That directness is also the security surface: if a permission expression is wrong, the client that connects straight to the DB is the attacker's entry point.

## Production Notes

**Licensing is the first gate, not a footnote.** BSL 1.1 permits self-hosting and embedding, but prohibits offering SurrealDB as a managed/commercial database service, and legal review is warranted before betting a product on it. This is a deliberate open-core stance (SurrealDB Cloud is the commercial offering) and disqualifies some use cases outright.

**Maturity and breaking changes.** The 1.x line and the jump to 2.x carried non-trivial SurrealQL and behavior changes; upgrades have historically required export/import migrations rather than in-place upgrades, and the on-disk format has evolved. Read the release notes and back up before upgrading — this is not yet a database where you upgrade blind.

**Single-node is the common, well-trodden path.** The distributed (TiKV/FoundationDB) topology is real but operationally heavy — you are now running a distributed KV cluster underneath SurrealDB. Most production users run a single node with RocksDB or SurrealKV plus their own backup/replication discipline. Durability guarantees depend on the chosen backend; SurrealKV's track record is comparatively short.

**Performance is model-flexibility, not raw-speed, oriented.** The multi-model engine and query planner are younger than Postgres/MongoDB equivalents; deep graph traversals, large `LIVE SELECT` fan-out, and complex permission expressions can be costly. Benchmark your own workload rather than assuming it will beat a tuned single-model store.

**"No backend code" cuts both ways.** Pushing auth and business rules into `PERMISSIONS`, events, and embedded JavaScript functions removes a server tier but concentrates correctness in SurrealQL definitions that are easy to get subtly wrong and harder to unit-test than application code.

## When to Use / When Not

**Use when:**
- You genuinely need several models (documents + graph + relational) behind one query language and want to avoid stitching multiple stores together.
- You want embedded/edge deployment (in-app, WASM, single binary) with realtime queries and auth built in.
- You're building a prototype or app backend and value developer velocity over a long durability track record.
- The BSL terms are compatible with how you ship (self-hosted app, not a competing DB service).

**Avoid when:**
- You require OSI-approved open source or must avoid source-available licensing constraints.
- You need a decade-proven durability and operational track record (banking-grade systems of record).
- Your workload is single-model and latency-critical — a tuned Postgres, MongoDB, or Neo4j will be more predictable.
- You want a fully managed ecosystem of third-party tools, hosted providers, and hiring pool that a mature database offers.

## Alternatives

- arangodb/arangodb — established multi-model (document + graph + key-value); use when you want the same "one engine, many models" idea with a longer track record.
- neo4j/neo4j — use when the graph is your primary model and you need mature Cypher tooling and graph algorithms.
- supabase/supabase — use when you want the BaaS/realtime experience but on battle-tested Postgres with OSI-open components.
- pocketbase/pocketbase — use when you want a single-binary backend for a smaller app and don't need multi-model or clustering.
- geldata/gel (formerly EdgeDB) — use when you want a typed, graph-relational query language backed by Postgres durability.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2021-12-09 | Public development begins[^4]. |
| 1.0.0-beta | 2022 | Long public beta series; frequent breaking changes. |
| 1.0.0 | 2023-09 | First general-availability release[^2]. |
| 2.0.0 | 2024-09 | Major release; SurrealQL and behavior changes, migration required[^5]. |

## References

[^1]: SurrealDB README and features overview. https://github.com/surrealdb/surrealdb — repository description: "A scalable, distributed, collaborative, document-graph database, for the realtime web."
[^2]: SurrealDB 1.0 general availability announcement (September 2023). https://surrealdb.com/blog
[^3]: Business Source License 1.1 as applied by SurrealDB. https://github.com/surrealdb/license
[^4]: GitHub API `repos/surrealdb/surrealdb`, `created_at` 2021-12-09; ~32.7k stars, 1.3k forks, last push 2026-07-06 (fetched 2026-07-15).
[^5]: SurrealDB 2.0 release notes. https://surrealdb.com/releases

## Tags

rust, database, multi-model, graph-database, document-database, realtime, backend-as-a-service, surrealql, distributed, source-available, nosql, embedded-database
