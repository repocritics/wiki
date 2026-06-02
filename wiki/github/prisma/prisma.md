# prisma/prisma

Prisma — a next-generation ORM for Node.js + TypeScript. Type-safe queries via a generated client, schema-first migrations, multi-database support.

## What it is

A TypeScript ORM that generates a type-safe database client from a single `schema.prisma` file. Supports PostgreSQL, MySQL, MariaDB, SQLite, SQL Server, MongoDB, CockroachDB. The defining DX: write a schema, run `prisma generate`, get a typed client that catches query errors at compile time.

## Key features

- Schema-first: `schema.prisma` is the source-of-truth, generates types.
- Multi-database support: Postgres, MySQL, MariaDB, SQLite, SQL Server, MongoDB, CockroachDB.
- Type-safe query builder.
- Migrations (`prisma migrate dev` / `deploy`).
- Prisma Studio — GUI for browsing data.
- Prisma Accelerate (paid) for connection pooling.
- Apache 2.0 licensed.

## Tech stack

- TypeScript primary.
- Rust query engine (compiled to native binary per platform).
- Distributed via npm `prisma` + `@prisma/client`.

## When to reach for it

- You're a TypeScript team and want a type-safe ORM with good DX.
- You want schema-driven migrations as the source of truth.
- You want multi-DB portability.

## When *not* to reach for it

- You want raw SQL control — use Drizzle ORM or just SQL with a query builder.
- You want minimal abstraction — Drizzle / Kysely are lighter.
- You're allergic to a Rust-binary in your stack — Prisma ships per-platform binaries.

## Maturity signal

46k stars, 2.2k forks, Apache 2.0, actively maintained under Prisma Inc. 8+ years.

## Alternatives

- Drizzle ORM — closer to raw SQL, smaller.
- TypeORM — older Active Record alternative.
- Kysely — lightweight query builder, not ORM.

## Tags

typescript, orm, database, postgresql, mysql, mongodb, apache-license, prisma, nodejs
