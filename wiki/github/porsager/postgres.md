# porsager/postgres

> A Postgres client for Node/Deno/Bun/Cloudflare built entirely around SQL tagged template literals, with parameterized queries and pipelining as the default path.

[GitHub repo](https://github.com/porsager/postgres) ·
[License: Unlicense](https://github.com/porsager/postgres/blob/master/LICENSE)

## Overview

Postgres.js (the `postgres` npm package, by Rasmus Porsager) is a pure-JavaScript
PostgreSQL client with no native/C dependencies. Its defining choice is that the
entire query surface is a tagged template function: you write `` sql`select ...` ``
and interpolated values become bound parameters (`$1, $2, ...`) sent separately to
the server, so string interpolation never reaches the wire and SQL injection through
values is structurally prevented. Everything else — transactions, cursors, `COPY`
streams, `LISTEN/NOTIFY`, dynamic query fragments — hangs off that one `sql` object.

The project positions itself against `pg` (node-postgres), which for a decade was the
default. Postgres.js trades `pg`'s large callback-era ecosystem and battle-tested
pooler compatibility for a smaller, more modern surface: promises everywhere, prepared
statements and pipelining on by default, first-class Deno/Bun/Cloudflare support, and
an API that reads like SQL. The tension is that the elegant template API is also its
main sharp edge — the same `sql()` helper does values, identifiers, and bulk
insert/update depending on argument shape, and the default prepared-statement behavior
interacts badly with transaction-mode connection poolers (see Production Notes).

It is widely used as the underlying driver for higher-level tools — notably Drizzle
ORM's postgres-js adapter — rather than only as an application-facing client.

## Getting Started

```bash
npm install postgres
```

```js
import postgres from 'postgres'

// Falls back to standard PG* environment variables (PGHOST, PGUSER, ...)
const sql = postgres('postgres://user:pass@host:5432/db', { max: 10 })

// Values are parameterized — this is injection-safe as written:
const age = 30
const users = await sql`
  select name, age
  from users
  where age > ${ age }
`
// users is a Result array: [{ name: 'Walter', age: 80 }, ...]

// Identifiers and helpers use the same sql() function, argument-shape dependent:
const user = { name: 'Murray', age: 68 }
await sql`insert into users ${ sql(user, 'name', 'age') }`
// -> insert into users ("name", "age") values ($1, $2)

await sql.end()
```

## Architecture / How It Works

The core is lazy promise evaluation. A `` sql`` `` expression does not run when
constructed; it runs when `await`-ed (or when `.execute()` is called). This laziness is
what lets nested `` sql`` `` fragments be distinguished from the outer query — a fragment
passed into an interpolation slot is spliced into the parent's SQL and parameter list
rather than executed on its own. Dynamic query building ("append this `where` clause
only if a filter is set") is therefore just conditional interpolation of fragments, with
no string concatenation and no injection surface for the *values*.

Under the wire it speaks the Postgres frontend/backend protocol directly. Two protocol
modes matter: the **extended** query protocol (one statement, real bound parameters,
and named prepared statements — the default) and the **simple** query protocol
(multiple statements in one round-trip, no parameters), reachable via `` sql`...`.simple() ``.
By default queries are prepared and the statement is cached per physical connection, so
repeated queries skip re-parsing. Independent queries issued without awaiting in between
are **pipelined** onto the connection, which is a large part of the throughput story.

Connection handling is built in: `postgres()` returns a manager, not a single socket.
`max` bounds the pool; queries acquire a connection, run, and release. `sql.begin(fn)`
reserves one connection for the duration of a transaction and hands the callback a scoped
`sql`, with automatic `ROLLBACK` on throw and `SAVEPOINT` support. `sql.reserve()` pins a
connection manually. Streaming primitives (`.cursor()`, `.forEach()`, and `COPY ...`
exposed as Node `Readable`/`Writable`) let you process result sets without buffering the
whole thing in memory.

Type handling is deliberate: to avoid silent precision loss, `int8`/`bigint` and
`numeric` come back as JavaScript strings unless you register custom parsers, and custom
type serializers/parsers can be attached via the `types` option.

## Production Notes

- **PgBouncer / transaction-pooled connections.** The default of named prepared
  statements breaks under transaction-mode poolers (PgBouncer `pool_mode = transaction`,
  and the pooled endpoints of Supabase/PgBouncer-fronted setups) because a prepared
  statement created on one server-side session may not exist on the next. Set
  `prepare: false` in that configuration. This is the single most common Postgres.js
  production surprise.
- **Serverless / edge lifecycle.** In short-lived environments (Lambda, Cloudflare
  Workers) you generally want a small `max` (often `1`) and to think carefully about when
  connections open and close; a warm module-level `sql` reused across invocations behaves
  very differently from one created per request. Postgres.js supports Cloudflare Workers,
  Deno, and Bun, but each has its own socket lifecycle caveats.
- **Identifier injection via helpers.** `sql(obj)` with no explicit column list uses the
  object's keys as column names, and `sql(columnName)` emits an identifier. Passing
  user-controlled strings into those slots reintroduces an injection vector that the value
  path protects you from — always whitelist columns/tables you interpolate as identifiers.
- **`sql.unsafe(...)`** disables parameterization (and prepared statements) for the string
  you pass. It exists for genuinely dynamic DDL/queries, but any user input concatenated
  into it is a straightforward injection. Treat it as a last resort.
- **BigInt/numeric as strings.** Code that does arithmetic on a `count(*)` or a money
  column will get string concatenation, not addition, unless you cast or register a parser.
  This bites teams migrating from clients that coerced to `Number`.
- **Loose TypeScript result typing.** Row types are not inferred from your schema; you
  supply them as generics (`` sql<User[]>`...` ``) or get a permissive type. This is why
  many typed-SQL setups pair Postgres.js with Drizzle or Kysely rather than using its raw
  types directly.
- **Query cancellation is best-effort.** `.cancel()` opens a second connection to send a
  protocol-level cancel; under high load it can race and, in pathological cases, cancel a
  different query. Fine for long analytics queries, risky as a general timeout mechanism.

## When to Use / When Not

**Use when:**
- You want promise-native, modern Postgres access with pipelining and prepared statements
  without wiring them yourself.
- You target Deno, Bun, or Cloudflare Workers, where `pg`'s native path is awkward.
- You like writing real SQL and want safe dynamic query composition without an ORM.
- You are building on top of it (Drizzle adapter, thin data layer).

**Avoid when:**
- You are behind a transaction-mode pooler and don't want to reason about `prepare: false`
  and prepared-statement lifecycle.
- You need schema-inferred, end-to-end type safety out of the box (reach for an ORM/builder).
- You depend on `pg`'s broad ecosystem (specific plugins, ORMs pinned to it, decades of
  Stack Overflow answers).
- Your team wants migrations, a schema DSL, and a generated client — that is an ORM's job,
  not this driver's.

## Alternatives

- brianc/node-postgres — the mature `pg` standard; use when you need the widest ecosystem
  and friction-free behavior behind poolers.
- drizzle-team/drizzle-orm — typed query builder that commonly uses Postgres.js as its
  driver; use when you want schema-inferred types over the same wire.
- prisma/prisma — full ORM with migrations and a generated client; use when you want
  managed schema and less hand-written SQL.
- kysely-org/kysely — type-safe SQL query builder; use when you want typed queries but not
  a full ORM.
- neondatabase/serverless — HTTP/WebSocket Postgres driver for edge runtimes; use when raw
  TCP is unavailable or undesirable in serverless.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-11 | First public release of the tagged-template Postgres client[^1]. |
| 2.x | 2020–2021 | Broad adoption as a modern `pg` alternative; Deno support. |
| 3.0 | 2022 | Major release; API and internals overhaul, expanded runtime support[^2]. |
| 3.x | 2022–2026 | Ongoing 3.x line; Bun/Cloudflare support, fixes and type refinements[^2]. |

## References

[^1]: porsager/postgres repository — created 2019-11-22. https://github.com/porsager/postgres
[^2]: Postgres.js changelog. https://github.com/porsager/postgres/blob/master/CHANGELOG.md

## Tags

javascript, nodejs, deno, bun, postgresql, database-client, sql, driver, tagged-templates, orm-driver
