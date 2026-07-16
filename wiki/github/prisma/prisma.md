# prisma/prisma

> Schema-first ORM for Node.js and TypeScript: a declarative model DSL that generates a fully typed query client.

[GitHub repo](https://github.com/prisma/prisma) ·
[Official website](https://www.prisma.io) ·
[License: Apache-2.0](https://github.com/prisma/prisma/blob/main/LICENSE)

## Overview

Prisma is a TypeScript/Node.js ORM built around a single source of truth — the `schema.prisma` file — from which it generates a type-safe query client, drives migrations, and introspects existing databases[^1]. It supports PostgreSQL, MySQL, MariaDB, SQL Server, SQLite, MongoDB, and CockroachDB. Unlike traditional ORMs, Prisma does not map to hand-written model classes with decorators; you write models in a purpose-built DSL and run `prisma generate` to produce the client, so the types are always derived from the schema rather than maintained by hand.

The current `prisma/prisma` repo is the second-generation product (Prisma ORM, formerly "Prisma 2"), a rewrite of the original GraphQL-based Prisma 1 server. The defining tension is codegen-vs-flexibility: Prisma trades the fine-grained SQL control of a query builder for a smaller, safer, more discoverable API surface. You get autocompletion and compile-time safety across relations and partial selections, but you give up arbitrary SQL shapes, and for years you gave up database-side JOINs entirely (Prisma issued separate queries and stitched results in the client). That tradeoff — plus a Rust query-engine binary that shipped alongside your app — has driven most of the ecosystem's criticism, and most of Prisma's recent engineering has gone into unwinding it.

The repo is actively maintained (last push 2026-07-14) with a large open-issue count (~2,600), typical of a widely deployed ORM that touches seven databases and many runtime targets.

## Getting Started

```bash
npm install prisma --save-dev
npm install @prisma/client @prisma/adapter-pg
npx prisma init
```

Define models in `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated"
}

datasource db {
  provider = "postgresql"
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  author   User?  @relation(fields: [authorId], references: [id])
  authorId Int?
}
```

Generate the client and query with a driver adapter:

```ts
import { PrismaClient } from './generated/client'
import { PrismaPg } from '@prisma/adapter-pg'

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL })
const prisma = new PrismaClient({ adapter })

const users = await prisma.user.findMany({
  where: { posts: { some: { title: { contains: 'prisma' } } } },
  include: { posts: true },
})
```

Every schema change requires re-running `npx prisma generate` to update the generated types.

## Architecture / How It Works

Prisma has four moving parts[^1]:

1. **Prisma schema** (`schema.prisma`) — the declarative DSL. It defines data sources, generators, and models. It is parsed and validated by a Rust component (`prisma-schema-wasm`), which is why schema errors are precise but the DSL is not extensible with plugins.
2. **Prisma Client** — the generated, typed query builder. `prisma generate` reads the schema and emits TypeScript. The client API is intentionally not SQL: `findMany`, `create`, `update`, nested writes, and relation filters compose into a JSON-like query object.
3. **Prisma Migrate** — declarative migrations. You edit the schema; `prisma migrate dev` diffs it against database state and generates SQL migration files. It requires a **shadow database** to compute diffs safely.
4. **Prisma Studio** — a local GUI for browsing and editing rows.

The historically central — and controversial — internal was the **query engine**: a Rust binary that the JS client spawned or embedded, which received the client's abstract query, planned it, emitted SQL, executed it against a connection pool it managed itself, and returned results. This is why older Prisma deployments shipped a platform-specific engine binary (`binaryTargets` in the generator) and why serverless bundles were large and cold starts slow.

The current direction, visible in the README's default examples, removes that binary. The **queryCompiler** compiles queries to SQL in TypeScript (with a WASM component) instead of a native engine, and **driver adapters** (`@prisma/adapter-pg`, `@prisma/adapter-libsql`, `@prisma/adapter-d1`, etc.) hand connection management to standard JavaScript database drivers rather than Prisma's own Rust pool[^2]. New projects use the `prisma-client` generator (superseding `prisma-client-js`) and configure connections in `prisma.config.ts` rather than passing a `url` in the datasource block. This co-evolution — moving the query planner from Rust into JS/WASM and the connection pool out to native drivers — is the single biggest architectural story of the project's last few years, and it directly targets the serverless/edge deployment pain the Rust engine caused.

## Production Notes

**The engine binary was a deployment footgun.** On the classic (`prisma-client-js`, non-adapter) setup, the Rust query engine must match the deployment platform. Missing or mismatched `binaryTargets` (e.g. `debian-openssl-3.0.x` vs `linux-musl` on Alpine, or a specific Lambda runtime) produces "engine not found" failures at runtime that don't appear locally. Driver adapters remove this class of bug but require rewiring instantiation.

**Serverless connection exhaustion.** Each `new PrismaClient()` opens its own pool. In serverless/edge functions that scale to many concurrent instances, this exhausts database connections fast. Mitigations: a single module-scoped client reused across invocations, an external pooler (PgBouncer in transaction mode, though Prisma has had friction with prepared statements there), or Prisma's paid **Accelerate** proxy. This is a common production incident for teams moving Prisma to serverless.

**No JOINs by default, historically.** For most of its life Prisma fetched relations with separate queries and merged in the client, which can be slower than a single JOIN and surprises people reading the query log. Database-side JOINs arrived later behind the `relationJoins` preview feature and are not universal across every provider — verify per provider before assuming.

**Migrations need care.** `prisma migrate dev` is for local development only (it can reset the database on drift); production uses `prisma migrate deploy`, which applies committed migration files without diffing. The shadow-database requirement complicates managed databases where you can't freely create databases. Some schema changes Prisma cannot express require dropping to raw SQL in the migration file.

**Raw SQL escape hatches are weakly typed.** `$queryRaw` returns loosely typed results; **TypedSQL** was added to give type-safe raw queries but requires a separate authoring flow. If your workload is JOIN-heavy analytics or needs window functions/CTEs, you will spend meaningful time outside the typed client.

**Generated client size and the generate step.** The generated client can be large, and forgetting to re-run `prisma generate` after a schema or dependency change is a frequent source of stale-type confusion in CI and monorepos.

## When to Use / When Not

**Use when:**
- You want end-to-end type safety from schema to query results without hand-maintaining model types.
- Your access patterns are CRUD and moderate relations — the ergonomic majority of application code.
- You value migrations, introspection, and a GUI as one integrated toolchain.
- The team benefits from a discoverable, autocompleted API over writing SQL.

**Avoid when:**
- Your workload is JOIN-heavy, analytical, or needs fine SQL control (CTEs, window functions) — a query builder fits better.
- You need the smallest possible cold start / bundle on edge runtimes and don't want to adopt driver adapters yet.
- You want a thin, transparent SQL layer with no codegen step in the build.
- You need database features Prisma's DSL doesn't model and don't want frequent raw-SQL fallbacks.

## Alternatives

- drizzle-team/drizzle-orm — SQL-like typed query builder, no codegen step, closer to raw SQL; use when you want SQL control and a smaller runtime.
- kysely-org/kysely — type-safe SQL query builder with no schema DSL; use when you want to write SQL but keep type safety.
- typeorm/typeorm — decorator/class-based Data Mapper + Active Record; use when you prefer entity classes and a traditional ORM shape.
- mikro-orm/mikro-orm — TypeScript Data Mapper with Unit of Work and identity map; use when you need managed entity state and transactions-as-units.
- sequelize/sequelize — long-established JS ORM; use for legacy JS codebases where mature breadth matters more than type safety.

## History

| Version | Date | Notes |
|---------|------|-------|
| Prisma 1 | 2018 | GraphQL-based Prisma server (evolved from Graphcool); separate deployed binary. |
| 2.0 GA | 2021-04 | "Production Ready" rewrite: schema DSL, generated client, Migrate, Rust query engine[^1]. |
| 3.0 | 2021-09 | Referential actions; interactive transactions maturing. |
| 4.0 | 2022-06 | Type/constraint changes, metrics, further preview features. |
| 5.0 | 2023-07 | Performance overhaul; smaller/faster generated client. |
| 6.0 | 2024-11 | Driver adapters and `prisma-client` generator pushed toward stable; ESM improvements[^2]. |

## References

[^1]: Prisma documentation — "What is Prisma ORM" / components overview. https://www.prisma.io/docs/orm/overview/introduction/what-is-prisma
[^2]: Prisma documentation — Driver adapters and database drivers reference. https://www.prisma.io/docs/orm/overview/databases/database-drivers

## Tags

typescript, javascript, orm, database, sql, postgresql, mysql, code-generation, query-builder, database-migrations, nodejs, type-safety
