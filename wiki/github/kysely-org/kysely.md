# kysely-org/kysely

> A type-safe TypeScript SQL query builder that infers result types from a database schema you declare in TypeScript.

[GitHub repo](https://github.com/kysely-org/kysely) ·
[Official website](https://kysely.dev) ·
[License: MIT](https://github.com/kysely-org/kysely/blob/master/LICENSE)

## Overview

Kysely (pronounced "Key-Seh-Lee") is a SQL query builder for TypeScript, first
released as 0.9.0 in November 2021[^1] by Sami Koskimäki, who also authored the
Objection.js ORM[^2]. It occupies the middle ground between raw SQL drivers and
full ORMs: you write queries with a fluent builder that mirrors SQL clauses
(`selectFrom`, `where`, `innerJoin`, `insertInto`), and TypeScript infers the
exact shape and column types of each result row from a `Database` interface you
supply. It draws its API lineage from Knex.js[^3] but replaces Knex's untyped,
mutable builder with an immutable, fully-typed one.

The defining tension is that Kysely's type safety is only as good as the
`Database` interface you hand it. Kysely does not read your live schema or
validate anything at runtime — the types are a compile-time contract you assert.
If the interface drifts from the actual database (a migration ran, a column was
renamed), the compiler still reports green while queries fail at runtime. The
ecosystem answer is code generation (see Production Notes), but the manual
declaration remains the source of truth Kysely trusts.

Kysely is still pre-1.0 (0.29.x as of mid-2026)[^4] despite wide adoption and
production use. The maintainers treat minor version bumps as potentially
breaking, so the 0.x number reflects API-stability policy, not immaturity.

## Getting Started

```bash
npm install kysely pg          # PostgreSQL via node-postgres
```

```ts
import { Kysely, PostgresDialect } from "kysely";
import { Pool } from "pg";

// You declare the schema; Kysely infers everything from it.
interface Database {
  person: { id: number; first_name: string; age: number | null };
  pet: { id: number; owner_id: number; name: string };
}

const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool: new Pool({ /* ... */ }) }),
});

// result is typed { first_name: string; pet_name: string }
const rows = await db
  .selectFrom("person")
  .innerJoin("pet", "pet.owner_id", "person.id")
  .select(["person.first_name", "pet.name as pet_name"])
  .where("person.age", ">", 18)
  .execute();
```

## Architecture / How It Works

Every builder method returns a new, immutable builder — chaining does not mutate
the previous object, so intermediate queries can be safely reused and composed.
Under the hood each call appends to an internal operation node tree (an AST of
the SQL statement). Execution runs that tree through three dialect-supplied
pieces:

- **`QueryCompiler`** — walks the operation node tree and emits a SQL string
  plus a parameter array. Parameters are always bound, not interpolated, so
  values passed through the builder (and through the `sql` template tag) are
  parameterized by default.
- **`DialectAdapter`** — declares dialect capabilities (returning clauses,
  transaction isolation levels, `RETURNING` support, migration locking).
- **`Driver` / `DatabaseConnection`** — owns the actual client library (`pg`,
  `mysql2`, `better-sqlite3`, Tedious for MSSQL) and executes the compiled SQL.

The entire type story lives in the compiler-facing generic signatures; there is
no decorator metadata, no reflection, and no runtime schema object. This is why
Kysely tree-shakes well and ships with zero runtime dependencies of its own —
the type inference has no runtime cost, only a compile-time one.

**Plugins** transform the operation node tree before compilation and transform
rows after execution. The built-in `CamelCasePlugin` is the canonical example:
it lets you write `firstName` in TypeScript while the database uses
`first_name`, rewriting identifiers on the way down and result keys on the way
back up. **Escape hatches** exist for anything the type system cannot express:
the `sql` template tag drops to raw (still parameterized) SQL with an asserted
return type, and `DynamicModule` handles column/table references that are only
known at runtime[^5].

Schema migrations are supported through a `Migrator` plus a `SchemaModule`
(`db.schema.createTable(...)`), but Kysely runs migration files you author by
hand and does not diff schemas or autogenerate migrations the way Prisma does.

## Production Notes

**TypeScript compile performance is the main operational cost.** Kysely's
inference leans hard on conditional and mapped types. On large `Database`
interfaces or deeply nested queries (many joins, subqueries, `with` CTEs), the
TypeScript language server can slow noticeably in the editor and `tsc` times
grow. This is the most frequently reported friction in practice; mitigations are
breaking huge queries into composed helpers, keeping the `Database` interface
lean, and staying on a recent TypeScript.

**Keeping the `Database` interface honest is on you.** Because nothing checks it
against the real database, teams almost always generate it. `kysely-codegen`[^6]
introspects a live database into a `Database` type; `prisma-kysely`[^7]
generates it from a Prisma schema for teams that keep Prisma for migrations but
query with Kysely. Without one of these, schema drift produces green builds and
runtime failures.

**Connection pooling and transactions are the driver's responsibility.** Kysely
wraps whatever pool you pass (`pg.Pool`, `mysql2` pool). `db.transaction()`
acquires a dedicated connection for the callback; long-running or nested
transaction logic behaves according to the underlying driver, not Kysely.

**Dialect breadth is uneven.** PostgreSQL, MySQL, MSSQL, and SQLite ship in
core. Deno, Bun, Cloudflare D1/Workers, PGlite, and other environments are
served by community dialects of varying maturity[^8] — verify the specific
dialect's status before committing, since some trail core features.

**0.x semver.** Minor bumps (e.g. 0.27 → 0.28 → 0.29) have carried breaking type
and API changes. Pin exact versions and read release notes before upgrading;
"it's just a query builder" upgrades have broken builds via tightened inference.

## When to Use / When Not

**Use when:**
- You want to write SQL, not learn an ORM's abstraction, but still want the
  compiler to catch typos in table/column names and wrong result-type usage.
- You already own your schema and migration story (SQL files, Atlas, Prisma
  migrate) and only need a typed query layer on top.
- You target multiple JS runtimes (Node, Deno, Bun, Workers) and want a
  zero-runtime-dependency, tree-shakeable builder.

**Avoid when:**
- You want managed migrations, schema-as-source-of-truth, and relations resolved
  for you — that is an ORM's job (Prisma, TypeORM).
- Your team is uncomfortable maintaining or generating a `Database` type, or
  cannot tolerate the compile-time cost of heavy type inference.
- You need dynamic, runtime-shaped queries as the norm — the escape hatches work
  but you lose the type safety that is the reason to pick Kysely.

## Alternatives

- prisma/prisma — full ORM with its own schema language and migration engine;
  use it when you want the schema to be the source of truth and less hand-written SQL.
- drizzle-team/drizzle-orm — type-safe, SQL-like builder that also ships
  migrations and schema definition; use it when you want one tool for both querying and migrations.
- knex/knex — the untyped JavaScript ancestor Kysely is modeled on; use it for
  legacy codebases or when you don't need static types.
- typeorm/typeorm — decorator-based Active Record / Data Mapper ORM; use it when
  you want entity classes and relations over explicit SQL.
- mikro-orm/mikro-orm — Unit-of-Work / identity-map ORM; use it when you need
  change tracking and an entity-graph model rather than a query builder.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9.0 | 2021-11-23 | First tagged release[^1]. |
| 0.28.0 | 2025 | Long-lived 0.28 line; incremental dialect and type work. |
| 0.29.0 | 2026-05-08 | Latest minor line[^4]. |
| 0.29.3 | 2026-07-05 | Latest patch as of this writing[^4]. |

## References

[^1]: Kysely releases — earliest tag 0.9.0, 2021-11-23. https://github.com/kysely-org/kysely/releases
[^2]: Sami Koskimäki (koskimas), also author of Objection.js. https://github.com/koskimas
[^3]: README states Kysely is "Inspired by Knex.js." https://github.com/kysely-org/kysely
[^4]: Kysely releases list — 0.29.x is the current line (0.29.3, 2026-07-05). https://github.com/kysely-org/kysely/releases
[^5]: Kysely API docs — `Sql` template tag and `DynamicModule`. https://kysely-org.github.io/kysely-apidoc/
[^6]: kysely-codegen — generates the `Database` type from a live database. https://github.com/RobinBlomberg/kysely-codegen
[^7]: prisma-kysely — generates Kysely types from a Prisma schema. https://github.com/valtyr/prisma-kysely
[^8]: README lists Node, Deno, Bun, Cloudflare Workers, browsers, and PGlite among supported environments. https://github.com/kysely-org/kysely

## Tags

typescript, sql, query-builder, database, type-safe, postgresql, mysql, sqlite, mssql, orm-alternative, nodejs, deno
