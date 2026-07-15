# knex/knex

> A dialect-portable SQL query builder for Node.js — the fluent layer under Objection, Bookshelf, and MikroORM, not an ORM itself.

[GitHub repo](https://github.com/knex/knex) ·
[Official website](https://knexjs.org/) ·
[License: MIT](https://github.com/knex/knex/blob/master/LICENSE)

## Overview

Knex is a query builder: a fluent JavaScript API that compiles method chains into
dialect-specific SQL strings and executes them through the underlying driver. It
targets PostgreSQL, MySQL/MariaDB, SQLite3, MSSQL, CockroachDB, and Oracle from a
single API, and bundles the plumbing most apps re-implement — connection pooling,
transactions, a migration runner, a seed runner, and streaming[^1]. It was created
by Tim Griesser (tgriesser) in 2012; since ~2019 primary maintenance has been
carried by Igor Savin (kibertoad)[^2].

The deliberate scope boundary is the whole point: knex builds and runs SQL but does
not map rows to objects, track identity, manage relations, or cache. That is left to
ORMs layered on top — Objection.js, Bookshelf, and (historically) parts of the
Node ORM ecosystem sit directly on knex. You write SQL-shaped code and get SQL-shaped
results (plain arrays of plain objects), with portability across engines as the payoff.

The defining tension in 2026 is age versus incumbency. Knex predates TypeScript's
dominance and its type support was retrofitted onto a runtime-first design; the
generics are shallow and do not verify that a query matches your schema. Newer,
TypeScript-first builders (Kysely, Drizzle) infer result types from the query itself.
Knex remains extremely widely deployed and stable, but active feature development has
slowed to maintenance pace, and it reads increasingly as the safe legacy choice rather
than the forward-looking one.

## Getting Started

```bash
npm install knex sqlite3   # driver is a separate dep: pg | mysql2 | better-sqlite3 | oracledb | tedious
```

```js
const knex = require('knex')({
  client: 'sqlite3',
  connection: { filename: './data.db' },
  useNullAsDefault: true,
});

await knex.schema.createTable('users', (t) => {
  t.increments('id');
  t.string('name');
});

const [id] = await knex('users').insert({ name: 'Tim' });
const rows = await knex('users').where('id', id).select('*');

await knex.destroy();   // release the pool; a hung process usually means you forgot this
```

Migrations and seeds run through the CLI (`npx knex migrate:make`, `migrate:latest`,
`seed:run`) against a `knexfile.js` that holds per-environment config.

## Architecture / How It Works

A knex instance is a factory bound to one `client` (dialect) and one connection pool.
Calling `knex('table')` returns a **query builder** — a mutable object that accumulates
clauses as you chain methods. The builder is a thenable: awaiting it (or calling
`.then`) triggers compilation and execution. Nothing hits the database until you await.

Compilation is dialect-driven. Each client (`Client_PG`, `Client_MySQL2`,
`Client_SQLite3`, `Client_MSSQL`, `Client_Oracledb`, `Client_CockroachDB`) subclasses a
base and overrides SQL generation — identifier quoting, `LIMIT`/`OFFSET` syntax,
`RETURNING` support, upsert (`onConflict`) semantics, and type coercion. The same chain
produces `"id"` on Postgres and `` `id` `` on MySQL. Portability is real but leaky:
`RETURNING` works on Postgres/MSSQL/SQLite but not MySQL, JSON operators differ, and
`onConflict().merge()` maps to different native syntax per engine.

Connection pooling is delegated to **tarn.js** (default `min: 2, max: 10`). Every
query acquires a connection from the pool, runs, and returns it. Transactions
(`knex.transaction`) pin one connection for the callback's duration; all builders
created from the transaction object route to that pinned connection.

`knex.raw()` is the escape hatch for anything the builder cannot express. It supports
positional bindings (`?` for values, `??` for identifiers) that are passed to the
driver as parameters — the safe path. String interpolation into `raw()` is the primary
SQL-injection vector in knex codebases.

## Production Notes

**Query builders are mutable and this bites people.** Chaining returns the *same*
object, not a copy. A "base query" reused across branches accumulates every clause from
every branch. If you intend to fork a query, call `.clone()` first. This is the single
most common knex bug in review.

**TypeScript support is structurally weak.** Result types come from generics you supply
(`knex<User>('users')`), not from the query — select a subset of columns and the type
still claims the full row; join two tables and the shape is not inferred. Treat knex's
types as documentation, not verification. Teams that need real type safety on the query
increasingly reach for Kysely or Drizzle instead.

**Driver value coercion surprises.** The `pg` driver returns `BIGINT`, `NUMERIC`, and
`DECIMAL` as JavaScript **strings** to avoid precision loss; `int8` counts from
`count(*)` come back as strings unless you parse them. MySQL drivers have their own
`DECIMAL`-as-string and date/timezone behavior. Knex passes these through unchanged —
the coercion is the driver's, not knex's, and must be handled in app code.

**Pool exhaustion presents as timeouts, not errors.** Leaked connections (a transaction
that never commits/rolls back, an unawaited builder) drain the pool; new queries then
hang until `acquireTimeoutMillis` fires. Symptoms look like a slow database. Always
`await` transaction callbacks and let them throw to auto-rollback.

**Migrations use a lock table.** The runner takes a lock (`knex_migrations_lock`) so
concurrent deploys don't double-run. A crashed migration can leave the lock stuck,
blocking further migrations until manually cleared (`migrate:unlock`). Migrations run
in filename order — the timestamp prefix is load-bearing.

**`.first()` and empty results.** `.first()` returns `undefined` (not `null`, not a
throw) when no row matches; forgetting the undefined check is common. `whereIn` with an
empty array resolves to a no-match condition rather than erroring.

## When to Use / When Not

**Use when:**
- You want to write SQL-shaped queries but keep them portable across Postgres/MySQL/SQLite.
- You need migrations, seeds, pooling, and transactions without adopting a full ORM.
- You're on an existing knex/Objection/Bookshelf stack — it is stable and battle-tested.
- Your team thinks in SQL and wants a thin, predictable layer over it.

**Avoid when:**
- Type-safe queries matter and you're greenfield — Kysely or Drizzle infer result types knex cannot.
- You want entity mapping, relations, and identity tracking out of the box — that's an ORM's job.
- You target a single database and can use its native/typed client without portability cost.
- You need cutting-edge dialect features quickly — maintenance-pace development means new SQL features land slowly.

## Alternatives

- kysely-org/kysely — TypeScript-first query builder; the query's result type is inferred from its columns. Use instead when static type safety on queries is the priority.
- drizzle-team/drizzle-orm — schema-in-TS ORM/builder with typed results and lightweight runtime. Use when you want types plus a thin ORM in one library.
- prisma/prisma — schema-DSL ORM with generated client and its own migration engine. Use when you want a managed, opinionated data layer over hand-written SQL.
- Vincit/objection.js — full ORM built directly on knex. Use when you like knex but also want relations, eager loading, and models.
- sequelize/sequelize — long-standing ORM with its own builder. Use when you want a mature all-in-one ORM rather than a query builder.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2013–2021 | Long-lived pre-1.0 line; production-used at 0.21/0.95 for years[^1]. |
| 1.0.0 | 2022-01 | First stable major after ~9 years on 0.x[^3]. |
| 2.0.0 | 2022-05 | Dropped older Node/driver support; API cleanups[^3]. |
| 3.0.0 | 2023-10 | Node 16+ baseline; dependency and dialect updates[^3]. |

## References

[^1]: knex README and guide — dialects, pooling, transactions, streaming. https://knexjs.org/
[^2]: knex maintainer contact (Igor Savin / @kibertoad) per project README. https://github.com/knex/knex
[^3]: knex releases and changelog. https://github.com/knex/knex/releases

## Tags

javascript, typescript, nodejs, sql, query-builder, database, orm, postgresql, mysql, sqlite, migrations, mssql
