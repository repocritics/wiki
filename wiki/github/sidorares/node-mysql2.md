# sidorares/node-mysql2

> A pure-JavaScript MySQL client for Node.js with server-side prepared statements — the de facto driver that most MySQL ORMs sit on top of.

[GitHub repo](https://github.com/sidorares/node-mysql2) ·
[Official website](https://sidorares.github.io/node-mysql2/) ·
[License: MIT](https://github.com/sidorares/node-mysql2/blob/master/License)

## Overview

MySQL2 is a MySQL/MariaDB protocol client for Node.js, published on npm as `mysql2`. It began as a continuation of `nodejs-mysql-native` by Andrey Sidorov: the wire-protocol parser was rewritten from scratch, and the public API was deliberately shaped to match the older, then-dominant `mysqljs/mysql` package (the `mysql` npm module)[^1]. The result is a near drop-in replacement — most code written for `mysql` runs on `mysql2` by changing the import — that adds the features the original never grew.

The defining addition is **server-side prepared statements** via `connection.execute()`. The legacy `mysql` driver only ever did client-side string interpolation through `query()`; MySQL2 keeps that path for compatibility but also speaks the binary prepared-statement protocol, which parameterizes queries on the server, caches the statement plan per connection, and returns typed binary results. On top of that it ships a Promise wrapper (`mysql2/promise`), connection pooling, compression, SSL, the `caching_sha2_password` auth plugin required by MySQL 8, non-UTF8 encodings, and even a partial MySQL *server* implementation and binary-log reader[^2].

Its practical importance is larger than its star count suggests: because the original `mysqljs/mysql` is effectively frozen, MySQL2 is the driver underneath Sequelize, Drizzle, Knex, TypeORM, and most other MySQL data layers in the Node ecosystem. For a large fraction of Node apps, MySQL2 is present transitively whether or not the author chose it directly.

## Getting Started

```bash
npm install --save mysql2
# TypeScript users also need Node types:
npm install --save-dev @types/node
```

```js
// Promise API + a pool — the recommended shape for servers
import mysql from "mysql2/promise";

const pool = mysql.createPool({
  host: "127.0.0.1",
  user: "app",
  password: process.env.DB_PASSWORD,
  database: "shop",
  connectionLimit: 10,
});

// execute() → server-side prepared statement (parameters bound on the server)
const [rows] = await pool.execute(
  "SELECT id, name FROM users WHERE age > ? LIMIT ?",
  [18, 5]
);
console.log(rows);
```

`query()` is the client-interpolated path (accepts `??`/`?` identifier and value escaping, multiple statements if enabled); `execute()` is the prepared-statement path. Prefer `execute()` for anything with user input.

## Architecture / How It Works

MySQL2 is pure JavaScript with no native bindings, so it installs cleanly on Linux, macOS, and Windows without a compiler[^2]. The core is a hand-written parser for the MySQL client/server protocol: a `PacketParser` reassembles TCP chunks into protocol packets, and command objects (Query, Execute, Prepare, Ping, Quit, and the auth handshake) drive a state machine over the socket.

Two query paths coexist and behave differently:

- **`query()`** — the text protocol. Parameters are escaped and interpolated into the SQL string client-side, then sent as one text query. Results come back as text and are parsed to JS types. This is the `mysqljs/mysql`-compatible path.
- **`execute()`** — the binary protocol. The driver sends a `COM_STMT_PREPARE`, receives a statement handle plus column metadata, then sends `COM_STMT_EXECUTE` with typed parameters. Prepared statements are cached per connection (keyed by SQL text) so repeated `execute()` of the same SQL reuses the handle.

Type decoding is where most surprises live. To avoid silent precision loss, `DECIMAL` and `BIGINT` values that exceed JS safe-integer range are returned as **strings** unless you opt into `decimalNumbers`, `supportBigNumbers`, and `bigNumberStrings`. `DATE`/`DATETIME` decoding is affected by the connection `timezone` and `dateStrings` options. A `typeCast` hook lets you override per-column decoding.

The pool (`createPool`) is a simple queue of connections with `connectionLimit`, `queueLimit`, optional `enableKeepAlive`, and idle handling. `PoolCluster` adds read/write splitting across hosts. Because prepared-statement caches are per *connection*, a pooled `execute()` only benefits from the cache when the same physical connection is reused for the same SQL.

## Production Notes

**Pooled prepared statements are re-prepared per connection.** Each connection in a pool maintains its own statement cache, so under a pool of N connections the first `execute()` of a given SQL on each connection pays a prepare round-trip. The cache is also bounded — churning through many distinct SQL strings on one connection evicts older statements. High-cardinality dynamic SQL can defeat the benefit entirely; for those, `query()` is sometimes simpler.

**MySQL 8 auth is the most common connection failure.** MySQL 8 defaults users to `caching_sha2_password`. Over a plaintext socket, MySQL2 must either use TLS or perform an RSA public-key exchange; misconfiguration surfaces as auth errors or `ER_NOT_SUPPORTED_AUTH_MODE`. The fixes are to enable SSL, pass the server's public key, or (last resort) alter the user to `mysql_native_password`.

**TLS with self-signed certs.** `ssl: { rejectUnauthorized: false }` disables verification and is widely copy-pasted; prefer supplying the CA (`ssl: { ca }`) so the connection is actually authenticated. Managed providers (PlanetScale, RDS, Aiven) each document their CA bundle.

**BIGINT/DECIMAL as strings** trips up developers expecting numbers. Aggregates like `COUNT(*)` return `BIGINT`; JSON-serializing rows can leak string-vs-number inconsistencies to clients. Decide the `supportBigNumbers`/`decimalNumbers` policy once, at pool creation.

**`LIMIT ?` and identifier binding.** Prepared statements bind *values*, not identifiers — you cannot parameterize table/column names via `execute()`. Placeholders for `LIMIT`/`OFFSET` work against modern MySQL but historically caused friction; validate on your server version.

**Connection lifetime.** Long-idle pooled connections can be killed server-side (`wait_timeout`) or by a load balancer, producing `PROTOCOL_CONNECTION_LOST` on next use. Use `enableKeepAlive` and expect to handle reconnection at the pool level; MySQL2 does not transparently retry a mid-flight query.

**Version note:** MySQL2 3.x is the current major line and where compatibility with recent MySQL/MariaDB auth and TLS behavior lives; staying on old 1.x/2.x releases against a MySQL 8 server is a frequent source of auth breakage.

## When to Use / When Not

**Use when:**
- You need a maintained MySQL/MariaDB driver for Node — this is the default choice, and `mysqljs/mysql` is essentially frozen.
- You want server-side prepared statements, a Promise API, pooling, and MySQL 8 auth without pulling in an ORM.
- You are choosing the driver under an ORM (Sequelize/Drizzle/Knex) — they target MySQL2's dialect.

**Avoid when:**
- You want a schema-first ORM with migrations and type generation rather than raw SQL — reach for a layer on top, not the driver alone.
- You need MariaDB-specific protocol features or its official connector's performance characteristics — evaluate the MariaDB driver.
- You want the MySQL X DevAPI / document-store protocol — that is Oracle's Connector/Node.js, not this project.

## Alternatives

- mysqljs/mysql — the original callback driver MySQL2 is compatible with; minimally maintained. Use only when you need the exact legacy API and no prepared statements.
- mariadb-corporation/mariadb-connector-nodejs — MariaDB's official driver; use when targeting MariaDB and wanting its bulk-insert/streaming and protocol tuning.
- mysql/mysql-connector-nodejs — Oracle's official Connector/Node.js with the X DevAPI; use when you want the document store or X Protocol.
- prisma/prisma — schema-first ORM with its own query engine; use when you want migrations and generated types instead of raw SQL.
- drizzle-team/drizzle-orm — typed SQL query builder that runs on MySQL2; use when you want type-safe queries but a thin, SQL-shaped API.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2013-04 | Rewrite continuing `nodejs-mysql-native`; parser rebuilt, API aligned to `mysqljs/mysql`[^1]. |
| 2.x | ~2019 | Promise wrapper, pooling, and broad `mysql` API parity established. |
| 3.0 | 2023-01 | Current major line; MySQL 8 `caching_sha2_password`, TLS, and bundled TypeScript types matured here[^3]. |

## References

[^1]: node-mysql2 README, "History and Why MySQL2" — continuation of MySQL-Native, API matched to mysqljs/mysql. https://github.com/sidorares/node-mysql2#history-and-why-mysql2
[^2]: node-mysql2 documentation (features: prepared statements, promise wrapper, pooling, SSL, compression, MySQL server, binary log). https://sidorares.github.io/node-mysql2/docs
[^3]: `mysql2` on npm (release history and current major version). https://www.npmjs.com/package/mysql2

## Tags

nodejs, javascript, typescript, mysql, mariadb, database-driver, sql, prepared-statements, connection-pool, orm-backend
