# FerretDB/FerretDB

> A MongoDB wire-protocol proxy that stores data in PostgreSQL — an Apache-2.0 answer to MongoDB's SSPL relicense.

[GitHub repo](https://github.com/FerretDB/FerretDB) ·
[Official website](https://www.ferretdb.com) ·
[License: Apache-2.0](https://github.com/FerretDB/FerretDB/blob/main/LICENSE)

## Overview

FerretDB is a proxy that speaks the MongoDB 5.0+ wire protocol on the front and
PostgreSQL on the back. Applications connect with an unmodified MongoDB driver
and issue BSON commands; FerretDB translates them into SQL against a Postgres
instance and returns MongoDB-shaped responses. The project exists because
MongoDB relicensed its server under the SSPL in 2018, which is not an
OSI-approved open-source license and is unusable for many redistributed and
managed-service use cases[^1]. FerretDB targets the large fraction of MongoDB
users who want document semantics and driver compatibility but do not need
MongoDB's proprietary advanced features.

The defining fact about FerretDB is that version 2 (2025) is a near-total
rearchitecture from version 1. Both versions are proxies, but v1 shipped its own
hand-written translation engine over plain PostgreSQL/SQLite tables, while v2
delegates document storage and query execution to Microsoft's DocumentDB
PostgreSQL extension — the same BSON-in-Postgres engine Microsoft open-sourced
from Azure Cosmos DB for MongoDB[^2]. This narrowed the compatibility and
performance gap substantially but also changed FerretDB's deployment story: you
now need a Postgres carrying a specific extension, not any stock Postgres. The v1
line lives on the `main-v1` branch and is effectively legacy[^3].

## Getting Started

The evaluation image bundles FerretDB, a preconfigured Postgres with the
DocumentDB extension, and `mongosh` in one container. It is for experiments only
— it keeps data inside the container and loses it on shutdown[^4].

```sh
docker run -d --rm --name ferretdb -p 27017:27017 \
  -e POSTGRES_USER=user \
  -e POSTGRES_PASSWORD=pass \
  ghcr.io/ferretdb/ferretdb-eval:2
```

```sh
# Connect with any MongoDB client; the driver cannot tell it is not MongoDB
mongosh "mongodb://user:pass@127.0.0.1:27017/"
```

```js
// Standard MongoDB CRUD works unchanged
db.orders.insertOne({ item: "widget", qty: 5, tags: ["a", "b"] })
db.orders.find({ qty: { $gte: 3 } })
```

For anything real, run FerretDB and a persistent Postgres-with-DocumentDB
separately via the production images or the Go embeddable package[^5].

## Architecture / How It Works

FerretDB is a stateless translation layer; all durable state lives in Postgres.

1. **Wire protocol front end.** FerretDB listens on port 27017 and decodes the
   MongoDB wire protocol and BSON. Drivers negotiate as they would with a real
   `mongod`, including the handshake that reports server version and capabilities.
2. **Query translation.** Incoming commands (`find`, `aggregate`, `update`,
   index operations, etc.) are converted into SQL and function calls.
3. **DocumentDB extension (v2).** Rather than reimplementing operators, v2 routes
   document operations to the DocumentDB Postgres extension, which stores BSON
   natively and provides the operator and index semantics. FerretDB is the wire
   gateway; the extension is the query engine[^2].

This is the central coupling story. In v1, FerretDB owned the SQL generation and
could run on any PostgreSQL (and SQLite); compatibility gaps were FerretDB's own
to close, one operator at a time. In v2, correctness and speed of the document
engine are inherited from DocumentDB, but you must run a Postgres that has the
extension installed — a stock managed Postgres without it will not work. FerretDB
Inc. and Microsoft collaborate on the extension, which was later contributed to
the Linux Foundation[^2].

Because the backend is Postgres, operational primitives — replication, backups,
connection pooling, disk-level encryption, monitoring — are Postgres's, not a
bespoke storage engine's. That is the pitch: reuse the mature Postgres operational
ecosystem while presenting a MongoDB API.

## Production Notes

- **Not a 100% MongoDB.** FerretDB implements a subset of the MongoDB surface and
  publishes a list of known differences and supported commands[^6]. Test your
  actual query and index patterns; drivers connecting successfully does not imply
  every aggregation stage, operator, or admin command behaves identically.
- **v1 → v2 is a migration, not an upgrade.** The storage layout and backend
  changed. Moving from a v1 deployment to v2 means a data migration, not a binary
  swap. Treat the `main-v1` branch as maintenance-only.
- **The extension is a hard dependency in v2.** You cannot point v2 at an
  arbitrary Postgres. Managed-Postgres users must confirm the DocumentDB
  extension is available, which many hosted providers do not yet offer — hence
  the roster of dedicated FerretDB Cloud / Civo / Tembo / Elestio offerings[^7].
- **Don't build it yourself.** Maintainers explicitly advise against building
  from source; generated files and build tags make the official binaries, Docker
  images, and packages the supported path[^8].
- **No transparent MongoDB clustering.** Sharding, replica-set elections, and
  MongoDB's operational topology do not carry over; you scale and replicate at the
  Postgres layer instead. Design HA around Postgres, not around `rs.status()`.
- **Version reporting.** FerretDB reports a MongoDB-compatible version in its
  handshake so drivers proceed; do not read that number as a guarantee of feature
  parity with that MongoDB release.

## When to Use / When Not

**Use when:**
- MongoDB's SSPL license blocks you (redistribution, managed offerings,
  license-averse organizations) and you want an Apache-2.0 document API.
- You already run and trust PostgreSQL and want one operational substrate.
- Your workload uses mainstream document CRUD, indexing, and aggregation rather
  than MongoDB's proprietary edges.

**Avoid when:**
- You depend on MongoDB features outside FerretDB's supported set (Atlas Search,
  Change Streams parity, specific aggregation operators, native sharding).
- You need a drop-in with zero validation — you must verify your query surface.
- You cannot provision a Postgres carrying the DocumentDB extension.

## Alternatives

- mongodb/mongo — the original; choose it when SSPL is acceptable and you want
  full first-party feature parity and Atlas.
- microsoft/documentdb — the underlying Postgres extension; use it directly when
  you want BSON-in-Postgres without a MongoDB wire gateway in front.
- postgres/postgres with JSONB — use plain Postgres when you can write SQL and do
  not need MongoDB driver compatibility at all.
- pingcap/tidb — use when you want a distributed SQL database with horizontal
  scaling rather than a MongoDB-compatible document layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| MangoDB (rename to FerretDB) | 2021-11 | Project starts as an SSPL response; renamed to FerretDB shortly after[^1]. |
| 1.0 GA | 2023-04 | First production release; own translation engine over PostgreSQL/SQLite backends. |
| 2.0 | 2025 | Rearchitected onto Microsoft's DocumentDB PostgreSQL extension; v1 moved to `main-v1`[^2][^3]. |

## References

[^1]: FerretDB — "Why do we need FerretDB?" and MongoDB SSPL background. https://www.ferretdb.com/sspl
[^2]: Microsoft DocumentDB PostgreSQL extension, used by FerretDB v2 as its engine. https://github.com/documentdb/documentdb
[^3]: FerretDB v1 legacy branch. https://github.com/FerretDB/FerretDB/tree/main-v1
[^4]: FerretDB README — Quickstart (evaluation image caveats). https://github.com/FerretDB/FerretDB
[^5]: FerretDB Go embeddable package. https://pkg.go.dev/github.com/FerretDB/FerretDB/v2/ferretdb
[^6]: FerretDB docs — migration compatibility, known differences and supported commands. https://docs.ferretdb.io/migration/compatibility/
[^7]: Managed FerretDB providers (FerretDB Cloud, Civo, Tembo, Elestio, Cozystack). https://cloud.ferretdb.com/
[^8]: FerretDB README — "Building and packaging" (advises using provided binaries). https://github.com/FerretDB/FerretDB

## Tags

go, database, document-database, mongodb-alternative, postgresql, proxy, wire-protocol, bson, open-source, sspl
