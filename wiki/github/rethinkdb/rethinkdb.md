# rethinkdb/rethinkdb

> A distributed JSON document database whose defining feature is changefeeds — queries that stay open and push results as the data changes.

[GitHub repo](https://github.com/rethinkdb/rethinkdb) ·
[Official website](https://rethinkdb.com) ·
[License: Apache-2.0](https://github.com/rethinkdb/rethinkdb/blob/main/LICENSE)

## Overview

RethinkDB is a distributed, schemaless document database that stores JSON and is queried through ReQL, a query language embedded as a fluent method chain in the host language rather than as SQL strings. Its distinguishing capability is the *changefeed*: appending `.changes()` to most queries turns them into a long-lived subscription that streams every subsequent insert, update, and delete matching the query. This inverts the usual polling model — instead of asking the database "what changed?" on a timer, the application declares interest once and the database pushes deltas as they happen[^1].

The project has an unusual history that dominates any adoption decision. RethinkDB Inc. (founded 2009 by Slava Akhmechet and Michael Glukhovsky) built the database as a venture-funded startup, shut the company down in October 2016, and published a widely-read postmortem on why the business failed despite the technology being well-regarded[^2]. In early 2017 the source was relicensed under Apache 2.0 and the assets were donated to the Linux Foundation, with the CNCF funding the transition[^3]. Development since then has been community-maintained and slow: real work still lands, but there is no full-time engineering team and releases are infrequent.

The defining tension is therefore not technical but organizational. RethinkDB solved the realtime-push problem cleanly years before mainstream databases added comparable features (MongoDB change streams, Postgres logical replication tooling, Firestore). What it cannot offer is the maintenance velocity, security-response cadence, and hiring pool of an actively-funded database. Teams choose it for the developer experience and accept the sustainability risk, or they choose a busier ecosystem and give up the ergonomics.

## Getting Started

Install a prebuilt package (macOS via Homebrew shown; Linux packages and a source build via `./configure --allow-fetch && make` are also supported):

```bash
brew install rethinkdb
rethinkdb            # starts a single node; web admin on http://localhost:8080
```

A minimal changefeed with the JavaScript driver:

```javascript
const r = require("rethinkdb");

const conn = await r.connect({ host: "localhost", port: 28015 });
await r.dbCreate("blog").run(conn).catch(() => {});
await r.db("blog").tableCreate("posts").run(conn).catch(() => {});

// Open a changefeed: this cursor stays alive and yields every future change.
const cursor = await r.db("blog").table("posts").changes().run(conn);
cursor.each((err, change) => {
  // change = { old_val, new_val }; old_val null on insert, new_val null on delete
  console.log(change);
});

// In another connection, this insert is pushed to the feed above.
await r.db("blog").table("posts").insert({ title: "hello" }).run(conn);
```

## Architecture / How It Works

RethinkDB is written in C++ and ships as a single `rethinkdb` binary that is both server and clustering agent. A cluster is formed by pointing nodes at each other with `--join`; there is no separate config server or router process the way sharded MongoDB requires.

**Sharding and replication.** Tables are range-sharded on the primary key. Each shard has a configurable number of replicas, and one voting replica per shard acts as primary for writes. Cluster metadata and per-shard primary election use Raft, added in the 2.1 release to provide automatic failover — if a primary's node dies, the remaining voting replicas elect a new one without operator intervention[^4]. This is why single-replica tables have no failover: there is nothing to elect.

**Storage engine.** RethinkDB uses its own log-structured, copy-on-write B-tree storage engine backed by a page cache (jemalloc-allocated), not an off-the-shelf engine like RocksDB or WiredTiger. Writes are durable by default (fsync on acknowledgement), with a per-write `durability` option to trade safety for throughput.

**ReQL.** Queries are built as an AST in the driver, serialized, and executed server-side. Because the language is host-language method chaining, it composes with normal control flow and supports server-side operations most drivers can't express as strings — subqueries, joins, `map`/`reduce`, and geospatial and date functions all run in the cluster. Secondary indexes (including compound, multi, and arbitrary-expression indexes) must be created explicitly; ReQL will otherwise fall back to full table scans.

**Changefeeds** are the same query machinery kept open. The server tags relevant write paths so that a committed change is matched against active feeds and pushed to subscribed cursors. Feeds can include the initial result set (`includeInitial`), collapse rapid updates (`squash`), and follow ordered/limited queries — but they are a live view, not a durable, replayable event log.

## Production Notes

**Sustainability is the first-order risk.** There is no vendor SLA and no guaranteed security-patch turnaround. For a datastore holding production data, factor in that a CVE may sit unpatched longer than it would for a funded database, and that operational knowledge lives mostly in old blog posts and a quiet community.

**Changefeeds are not a message queue.** They deliver current state transitions to *connected* subscribers. A client that disconnects misses everything during the gap; there is no offset or cursor to resume from. Under heavy write load `squash` may coalesce intermediate values, so a feed is not guaranteed to show every distinct version of a row. If you need exactly-once, replayable, or durable event delivery, put Kafka/NATS in front of or alongside RethinkDB rather than treating feeds as the log of record.

**Indexes decide performance.** `getAll` and `between` are fast only against an index; forgetting to create the secondary index silently produces a table scan that looks fine in dev and collapses at scale. The web admin and `.info()` help confirm index usage. Changefeeds on ordered/limited queries have additional index requirements.

**Failover requires an odd voting quorum.** Automatic failover needs at least three voting replicas per shard to tolerate one loss while keeping a majority. Two-replica setups do not fail over usefully. Non-voting replicas add read capacity but do not participate in election.

**Memory and cache.** The cache size defaults to a fraction of RAM and is set per process with `--cache-size`; co-locating RethinkDB with other memory-hungry services without capping it is a common cause of OOM. There are default limits on array size and document depth in ReQL (configurable via `arrayLimit`) that surface as errors on large aggregations.

**Upgrades.** Cross-version upgrades between minor releases are generally in-place, but always take a `rethinkdb dump` (a client-driver-based logical backup) first — it is the supported backup path and the safety net if an on-disk format assumption changes. Given the slow release cadence, most clusters run whichever 2.4.x they were installed with for years.

## When to Use / When Not

**Use when:**
- You want realtime push (live dashboards, collaborative apps, presence, feeds) and value the ergonomics of declaring a subscription in one line.
- Your data is naturally document-shaped JSON and you want ReQL's expressive server-side queries and joins.
- You can accept a community-maintained datastore and are willing to own operations and patch risk yourself.

**Avoid when:**
- You need a vendor with guaranteed security response, long-term roadmap, and a large hiring pool — the post-2016 maintenance reality disqualifies it for many risk-averse orgs.
- You need durable, replayable event streaming with delivery guarantees — use a real log/broker instead of changefeeds.
- You want strong relational modeling, transactions across many documents, or SQL — a relational database fits better.

## Alternatives

- mongodb/mongo — the mainstream document database; change streams cover much of the changefeed use case with a far larger ecosystem and commercial support. Use when maintenance velocity and hiring matter more than ergonomics.
- surrealdb/surrealdb — actively developed multi-model database with live queries; the closest modern spiritual successor to RethinkDB's push model. Use when you want the realtime-query idea on a currently-funded project.
- apache/couchdb — HTTP/JSON document store with a `_changes` feed and offline-first replication. Use when replication and eventual sync between nodes/devices is the priority.
- supabase/supabase — Postgres plus realtime subscriptions and a managed platform. Use when you prefer relational modeling and SQL but still want change subscriptions.
- meteor/meteor — full-stack framework whose reactive data layer targets the same realtime-app problem at the application tier. Use when you want the realtime experience built into the app framework rather than the database.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012-11 | First numbered release; document store with the admin UI. |
| 2.0 | 2015-04 | Declared production-ready; ReQL geospatial support[^1]. |
| 2.1 | 2015-08 | Automatic failover via Raft-based consensus[^4]. |
| 2.3 | 2016-04 | User accounts and permissions, TLS/encryption. |
| — | 2016-10 | Company shut down; project future uncertain[^2]. |
| — | 2017 | Relicensed Apache 2.0, moved to the Linux Foundation / CNCF[^3]. |
| 2.4 | 2019-12 | First major community-driven release after the transition. |

## References

[^1]: RethinkDB documentation — introduction and ReQL overview. https://rethinkdb.com/docs/
[^2]: Slava Akhmechet, "Why RethinkDB failed" — 2017. https://www.defmacro.org/2017/01/18/why-rethinkdb-failed.html
[^3]: The Linux Foundation / CNCF, "The RethinkDB open source project moves to the Linux Foundation" — 2017. https://www.linuxfoundation.org/press/press-release/the-rethinkdb-open-source-project-moves-to-the-linux-foundation
[^4]: RethinkDB documentation — failover and clustering. https://rethinkdb.com/docs/failover/

## Tags

cpp, database, nosql, document-database, realtime, changefeeds, distributed-systems, json, reql, cncf
