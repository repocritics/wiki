# apache/couchdb

> A document database that speaks HTTP/JSON natively and treats replication — not sharding — as its defining primitive.

[GitHub repo](https://github.com/apache/couchdb) ·
[Official website](https://couchdb.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/couchdb/blob/main/LICENSE)

## Overview

CouchDB is a schema-free document store, written in Erlang, that exposes its
entire API over HTTP with JSON documents as the unit of storage. It was
started by Damien Katz in 2005, rewritten from C++ to Erlang, and entered the
Apache Incubator in 2008[^1]. Its distinguishing idea is that every node is a
full peer: the replication protocol is bidirectional, incremental, and
conflict-tolerant, which makes CouchDB unusually good at offline-first and
edge topologies where a client (or a whole datacenter) goes dark and later
reconciles.

The defining tension is between that replication-first design and everything
people expect from a modern database. CouchDB gives up ad-hoc SQL-style
querying (you get MapReduce views and the Mango declarative query language
instead), gives up strong global consistency (the cluster is eventually
consistent with quorum reads/writes), and historically shipped with a
famously insecure default — the "admin party," where an unconfigured node let
anyone do anything — removed only in 3.0 (2020)[^2]. What you get in return is
a database that a phone, a browser (via PouchDB), and a server can all run and
sync against using the same wire protocol.

CouchDB is a slow-moving, mature project. It is not where you go for the
newest query features; it is where you go when a multi-writer, sync-heavy data
model is the actual problem you have.

## Getting Started

```bash
# Docker is the least painful install path
docker run -d --name couchdb \
  -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=secret \
  -p 5984:5984 apache/couchdb:3
```

```bash
# Everything is HTTP. Create a database, insert a doc, read it back.
curl -X PUT http://admin:secret@127.0.0.1:5984/mydb

curl -X POST http://admin:secret@127.0.0.1:5984/mydb \
  -H 'Content-Type: application/json' \
  -d '{"type":"note","title":"hello","tags":["a","b"]}'
# => {"ok":true,"id":"<uuid>","rev":"1-<hash>"}

# Mango query — declarative, no view needed
curl -X POST http://admin:secret@127.0.0.1:5984/mydb/_find \
  -H 'Content-Type: application/json' \
  -d '{"selector":{"type":"note"},"limit":10}'
```

Every document carries an `_id` and a `_rev` (revision). Updates are
optimistic: you must send the current `_rev`, and a stale `_rev` gets a
`409 Conflict`. The admin UI, Fauxton, ships at `/_utils`.

## Architecture / How It Works

**Storage.** Each database is an append-only file backed by B+-trees. Writes
never mutate in place; they append a new revision and update the tree roots.
This gives crash-only semantics (recovery is just truncating a partial write)
and MVCC reads that never block writers, at the cost of file growth that
requires periodic **compaction** to reclaim space.

**Revisions are not version history.** The `_rev` chain exists to detect
conflicts during replication, not to let you time-travel. Old revisions are
discarded on compaction and must never be relied on as a history feature — a
common and costly misconception.

**Clustering (2.0+).** A CouchDB cluster shards each database into `q` shards
and replicates each shard `n` times (default `n=3`), coordinated by a
Dynamo-style quorum[^3]. Reads and writes take `r`/`w` quorum parameters. There
is no single primary; the cluster is eventually consistent, and a write
acknowledged by a quorum can still be temporarily invisible to a node outside
it.

**Views (MapReduce).** Secondary indexes are JavaScript map/reduce functions
evaluated by an external query server (historically SpiderMonkey). Views are
built lazily and incrementally: the index updates on read unless you run a
background indexer, so the first query after many writes can be slow.

**Mango.** A declarative JSON query language (`_find`) added to reduce the
need to hand-write views. It can use JSON indexes but silently falls back to
full scans when no index matches — a frequent performance trap.

**Replication.** The replicator is itself an HTTP client that streams the
`_changes` feed from a source and applies revisions to a target. The same
protocol powers PouchDB in the browser, cross-datacenter sync, and filtered
partial replication. Conflicts are stored, not resolved: both revisions are
kept, a deterministic winner is chosen, and the application is expected to
merge losers.

## Production Notes

**Compaction is operational work, not automatic magic.** Append-only files
grow without bound under write/update load. Auto-compaction exists but must be
tuned; unmanaged databases have run hosts out of disk. Budget for compaction
windows and free space (roughly 2x the live data size during compaction).

**View index rebuilds are expensive and easy to trigger.** Changing a single
character in a design document's map function invalidates the entire view and
forces a full rebuild across all shards. On large databases this can take
hours and blocks queries against that view. Treat design documents as
migrations, not casual edits.

**Reduce functions must actually reduce.** A reduce that does not shrink its
input (returning large objects, accumulating arrays) trips the "reduce_overflow"
protection or degrades to unusable performance. This is the single most common
self-inflicted view problem.

**The `_changes` feed is the integration point — and a footgun.** It is the
right way to build event-driven pipelines, but a continuous feed without a
persisted checkpoint (`since` sequence) will reprocess history on restart.
Sequence values are opaque and change format across major versions.

**Document-per-entity, not table-per-type.** Modeling relational data as many
small cross-referenced documents forces `_all_docs` fan-out or joins in the
application. CouchDB rewards denormalized, self-contained documents; teams that
port a normalized schema directly are usually disappointed.

**The abandoned 4.0.** For several years the roadmap targeted a 4.0 rebuilt on
FoundationDB as the storage layer; that effort was ultimately dropped and
development continued on the existing 3.x line[^4]. If you find old material
promising FoundationDB-backed CouchDB, it did not ship.

## When to Use / When Not

**Use when:**
- You need offline-first sync between clients and servers (mobile, field
  devices, browser apps) with automatic conflict capture.
- Your access pattern is "read/write whole documents by id," not ad-hoc joins.
- You want a database you can operate over plain HTTP with no client driver.
- Multi-master / multi-datacenter replication is a first-order requirement.

**Avoid when:**
- You need rich ad-hoc queries, aggregations, or joins — Mango and views are
  deliberately limited.
- You need strong, immediate global consistency across a cluster.
- Your data is highly relational and normalized.
- You want a fast-moving database with a large feature cadence; CouchDB is
  intentionally conservative.

## Alternatives

- pouchdb/pouchdb — not a competitor but the in-browser/Node counterpart that
  speaks the same replication protocol; use it when you want CouchDB sync on
  the client.
- couchbase/couchbase — shares ancestry and the name but diverged into a
  memcached-derived KV/N1QL engine; use it when you need low-latency caching
  and SQL-like querying at scale rather than peer replication.
- mongodb/mongo — use instead when you want a document store with a rich query
  and aggregation language and don't need multi-master sync.
- rethinkdb/rethinkdb — use when realtime changefeeds/push queries are the core
  need, accepting a smaller and less active community.
- surrealdb/surrealdb — use when you want a newer multi-model database with
  query flexibility, if you can accept far less operational track record.

## History

| Version | Date | Notes |
|---------|------|-------|
| Incubation | 2008 | Enters Apache Incubator; graduates to top-level project[^1]. |
| 1.0 | 2010-07 | First stable release. Single-node, HTTP/JSON, MapReduce views. |
| 2.0 | 2016-09 | Clustering: sharding + quorum replication, Mango query, Fauxton UI[^3]. |
| 3.0 | 2020-02 | Secure by default (admin party removed), single-node default, partitioned databases[^2]. |
| 3.2 | 2021-10 | Performance and clustering improvements. |
| 3.3 | 2022-12 | Continued 3.x line after the FoundationDB 4.0 plan was dropped[^4]. |

## References

[^1]: Apache CouchDB project history. https://couchdb.apache.org/history.html
[^2]: CouchDB 3.0 release notes — secure by default, admin party removed. https://docs.couchdb.org/en/stable/whatsnew/3.0.html
[^3]: CouchDB clustering / sharding documentation. https://docs.couchdb.org/en/stable/cluster/sharding.html
[^4]: CouchDB developer mailing-list discussion on discontinuing the FoundationDB-based 4.0 effort. https://lists.apache.org/list.html?dev@couchdb.apache.org

## Tags

erlang, database, document-database, nosql, http-api, json, replication, offline-first, mapreduce, mango-query, eventual-consistency, apache
