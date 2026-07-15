# brianc/node-postgres

> The de facto PostgreSQL driver for Node.js — a pure-JavaScript wire-protocol client (npm `pg`) that most ORMs and query builders sit on top of.

[GitHub repo](https://github.com/brianc/node-postgres) ·
[Official website](https://node-postgres.com) ·
[License: MIT](https://github.com/brianc/node-postgres/blob/master/LICENSE)

## Overview

node-postgres, published on npm as `pg`, is the oldest and most widely deployed PostgreSQL client for Node.js, first committed in 2010[^1]. It speaks the PostgreSQL frontend/backend wire protocol directly in JavaScript, so it works with no compiled dependencies on Node, and increasingly on Bun, Deno, and Cloudflare Workers. An optional native binding (`pg-native`, wrapping libpq) shares the same API for workloads that want C-level parsing.

The project is deliberately low-level. It is a driver, not an ORM: you write SQL strings, pass parameters positionally, and get back rows. This is its defining tradeoff — it gives you almost nothing beyond connections, pooling, type coercion, and a query queue, and expects a higher layer (Drizzle, Prisma, Knex, Sequelize, or hand-written SQL) to provide ergonomics. Most of those higher layers use `pg` as their Postgres backend, which is why the package sees on the order of tens of millions of weekly npm downloads despite its small surface area.

At ~13k stars, ~1.3k forks, and commits landing as recently as mid-2026, it is actively but conservatively maintained by Brian Carlson with community contributions[^1]. The ~500 open issues reflect scope (a monorepo of seven packages) and age more than neglect; backwards compatibility is treated as close to sacred.

## Getting Started

```bash
npm install pg
```

```js
import { Pool } from "pg";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,                     // max clients in the pool
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
});

// Parameterized query — never string-concatenate user input
const { rows } = await pool.query(
  "SELECT id, email FROM users WHERE created_at > $1",
  [new Date("2026-01-01")]
);
console.log(rows);

// A transaction MUST run on a single checked-out client, not pool.query()
const client = await pool.connect();
try {
  await client.query("BEGIN");
  await client.query("UPDATE accounts SET balance = balance - $1 WHERE id = $2", [100, 1]);
  await client.query("UPDATE accounts SET balance = balance + $1 WHERE id = $2", [100, 2]);
  await client.query("COMMIT");
} catch (e) {
  await client.query("ROLLBACK");
  throw e;
} finally {
  client.release();
}
```

## Architecture / How It Works

node-postgres is a monorepo of small, single-purpose packages[^2]:

- **pg** — the public entry point. Exposes `Client` (one TCP connection) and `Pool`.
- **pg-protocol** — a from-scratch parser/serializer for the Postgres v3 wire protocol; rewritten in TypeScript around the 8.0 line, and the performance-critical core.
- **pg-pool** — the connection pool implementation surfaced as `pg.Pool`.
- **pg-cursor** / **pg-query-stream** — read large result sets incrementally instead of buffering every row into memory.
- **pg-connection-string** — parses `postgres://` URLs into config objects.
- **pg-native** — optional libpq bindings with an API-compatible `Client`.

A single `Client` is a single connection and is **not concurrent**: queries issued on it are serialized into an internal queue and executed one at a time. Concurrency comes from the `Pool`, which hands out idle clients and queues callers when all are busy. This is the most common source of confusion — `pool.query()` grabs a client, runs one statement, and returns it, which is fine for autocommit queries but wrong for multi-statement transactions.

Type coercion is pluggable via `pg-types`. Postgres sends everything as text (or binary) on the wire; `pg` maps OIDs to JS parsers. Two defaults trip people up: `bigint`/`int8` is returned as a **string** (to avoid silent precision loss past 2^53), and `numeric`/`decimal` likewise. `timestamp without time zone` is parsed against the process's local timezone. All of these are overridable through `pg.types.setTypeParser`.

Prepared statements are opt-in: pass a `name` to a query config and Postgres caches the plan for that connection. Named statements are per-connection, so in a pool the same plan may be re-prepared across different physical clients.

## Production Notes

- **Handle the pool's `error` event.** Idle pooled clients hold live TCP connections; if the server or a proxy (PgBouncer, RDS Proxy, a cloud load balancer) drops one, the client emits `error`. An unhandled `error` on an idle client will crash the Node process. Always attach `pool.on("error", ...)`.
- **No query timeout by default.** A hung query will occupy a pool client indefinitely. Set `statement_timeout` / `query_timeout` on the client config, and `connectionTimeoutMillis` so `pool.connect()` fails fast under saturation.
- **Pool sizing.** Default `max` is 10. More is not better — Postgres itself is connection-limited, and past a point you want a server-side pooler (PgBouncer in transaction mode) in front, with a smaller `pg` pool per instance. In serverless/edge, long-lived pools don't fit the execution model; use a pooler or a driver designed for it.
- **PgBouncer transaction mode breaks named prepared statements** and `LISTEN/NOTIFY`, because those need session affinity that transaction pooling doesn't provide. Know which pooling mode you're on.
- **bigint-as-string** surprises code that does arithmetic on `count(*)` or ID columns; decide explicitly whether to override the parser or keep strings.
- **SSL.** `ssl: { rejectUnauthorized: false }` is common in tutorials and disables certificate verification; supply the CA instead for real deployments.
- **pg-native** can be faster for parse-heavy workloads but adds a libpq/compiler build dependency and complicates Docker images and serverless bundles; the pure-JS path is the safe default and where most attention goes.

## When to Use / When Not

**Use when:**
- You want direct, predictable SQL access to Postgres with minimal abstraction.
- You're building the data layer that an ORM or query builder will sit on, or you prefer hand-written SQL.
- You need Postgres-specific features: `COPY`, `LISTEN/NOTIFY`, cursors, streaming, parameterized queries.

**Avoid when:**
- You want schema management, migrations, relations, and type-safe queries out of the box — reach for an ORM/query builder (which will use `pg` underneath anyway).
- You're on serverless/edge with per-request lifecycles and no external pooler — a pooler-aware or HTTP-based driver fits better.
- You want tagged-template query ergonomics and built-in `bigint`/pipelining conveniences — `porsager/postgres` is a leaner modern alternative.

## Alternatives

- porsager/postgres — modern tagged-template client, faster in many benchmarks, use when you want ergonomic SQL and minimal config over the mature-but-verbose `pg` API.
- drizzle-team/drizzle-orm — typed SQL-like query builder; use when you want type safety and migrations but still thin runtime (it can drive `pg`).
- prisma/prisma — full ORM with schema DSL and generated client; use when you want managed migrations and relations over raw SQL.
- knex/knex — SQL query builder; use when you want programmatic query construction across multiple SQL dialects.
- sequelize/sequelize — mature batteries-included ORM; use when you need Active Record–style models and broad DB support.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010-10-15 | First commit; pure-JS Postgres client[^1]. |
| 6.0 | 2016 | API cleanup; Promise support alongside callbacks. |
| 7.0 | 2017 | Pool rework, dropped legacy `pg.connect` patterns. |
| 8.0 | 2020 | pg-protocol rewritten in TypeScript; stricter defaults, dropped old Node versions[^2]. |

## References

[^1]: node-postgres repository and history, Brian Carlson. https://github.com/brianc/node-postgres
[^2]: Monorepo package layout (pg, pg-pool, pg-protocol, pg-native, pg-cursor, pg-query-stream, pg-connection-string). https://github.com/brianc/node-postgres/tree/master/packages
[^3]: Official documentation — pooling, queries, types, transactions. https://node-postgres.com

## Tags

javascript, typescript, postgresql, database, database-driver, connection-pooling, node, sql, orm-backend, libpq, wire-protocol
