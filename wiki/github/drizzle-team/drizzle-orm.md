# drizzle-team/drizzle-orm

> A thin, typed SQL layer for TypeScript — a query builder that calls itself an ORM, with zero runtime dependencies and no query engine.

[GitHub repo](https://github.com/drizzle-team/drizzle-orm) ·
[Official website](https://orm.drizzle.team) ·
[License: Apache-2.0](https://github.com/drizzle-team/drizzle-orm/blob/main/LICENSE)

## Overview

Drizzle is a TypeScript ORM for PostgreSQL, MySQL, and SQLite, first released publicly in 2022[^1]. Its defining choice is to stay close to SQL: you declare tables in TypeScript, and the query builder maps almost one-to-one onto the SQL you would otherwise write by hand. There is no separate schema language, no code generation step for the client, and no query engine — the published package is plain JavaScript with zero runtime dependencies, ~7.4 kB minified+gzipped and tree-shakeable[^2].

The audience is developers who want static type safety over their database without the abstraction distance of a full ORM. Two query styles coexist: a SQL-like builder (`db.select().from(users).where(...)`) that reads like the underlying statement, and a Relational Query Builder (`db.query.users.findMany({ with: { posts: true } })`) that returns nested objects across relations. The first is the honest core of the library; the second is a convenience layer with its own performance characteristics (see Production Notes).

The central tension is verbosity versus control. Drizzle deliberately does *not* hide SQL, so complex queries are as verbose as the SQL they represent — you trade Prisma-style terseness for the ability to see and shape every statement. It also ran on `0.x` version numbers for years despite heavy production use, signalling that its authors reserved the right to break APIs[^3]; treat pinned versions and changelogs as load-bearing.

## Getting Started

```bash
npm i drizzle-orm
npm i -D drizzle-kit          # migrations + studio CLI
```

```ts
// schema.ts
import { pgTable, serial, text, integer } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  name: text("name").notNull(),
  age: integer("age"),
});
```

```ts
// db.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { eq } from "drizzle-orm";
import { users } from "./schema";

const db = drizzle(process.env.DATABASE_URL!);

// SQL-like builder — types inferred from the schema, no codegen
const adults = await db
  .select()
  .from(users)
  .where(eq(users.age, 18));
```

Schema changes are managed with Drizzle Kit: `drizzle-kit generate` emits SQL migration files from the diff, `drizzle-kit migrate` applies them, and `drizzle-kit push` applies schema changes directly (intended for prototyping, not production)[^4]. `drizzle-kit studio` opens Drizzle Studio, a local database browser.

## Architecture / How It Works

Drizzle is a **dialect + driver** matrix, not a single client. You import a driver-specific `drizzle()` factory (`drizzle-orm/node-postgres`, `drizzle-orm/postgres-js`, `drizzle-orm/better-sqlite3`, `drizzle-orm/libsql`, `drizzle-orm/bun-sqlite`, `drizzle-orm/neon-http`, `drizzle-orm/planetscale`, `drizzle-orm/d1`, and others). Each wraps an existing database driver rather than shipping its own connection layer, which is why the package has zero dependencies and runs in Node, Bun, Deno, Cloudflare Workers, edge runtimes, and browsers — anywhere the underlying driver runs[^1].

Schema is defined with dialect-scoped builders (`pgTable`, `mysqlTable`, `sqliteTable`). These are plain objects that carry column type metadata at both the type level (for inference) and the value level (for query construction). There is no reflection against a live database and no generated client — the TypeScript compiler *is* the type system, so the accuracy of your types is only as good as your schema staying in sync with the real database.

Queries compile to parameterized SQL through a dialect-specific builder. The `sql` template tag is the documented escape hatch for anything the builder does not express, and it composes with typed fragments (`sql<number>\`count(*)\``). The Relational Query Builder sits on top: it takes a declarative `with` tree and compiles it into a *single* SQL statement using lateral joins and JSON aggregation, so one `findMany` with nested relations is one round-trip, not an N+1 cascade.

Drizzle Kit is a separate package and a separate mental model. It reads your TS schema, computes a diff against a stored snapshot, and emits raw SQL migrations. Because migrations are SQL files (not an opaque format), they are reviewable and editable — but the diff engine is heuristic, and renames, enum changes, and column type changes are the classic cases where it guesses wrong and prompts you.

## Production Notes

**TypeScript compiler pressure is the real cost.** Large schemas and deeply nested relational queries can slow `tsc` and editor responsiveness noticeably; inference over wide tables and many relations is where teams feel it. Splitting schema files and avoiding gratuitously deep `with` trees are the usual mitigations.

**The Relational Query Builder had genuine performance problems.** The v1 design generated large lateral-join + JSON-aggregation statements that some databases planned poorly, producing slow queries on nested reads. This drove a v2 redesign of the relational layer[^5]; if you rely on `db.query.*`, benchmark against the equivalent hand-written `select`/`join`, and be prepared to drop to the SQL-like builder for hot paths.

**`drizzle-kit push` is not a migration strategy.** It is convenient for local iteration but applies diffs without a reviewable migration history. Production workflows should use `generate` + `migrate` with the SQL files committed to the repo. Expect to hand-edit generated migrations for renames and non-trivial type changes — the diff engine cannot distinguish a rename from a drop+add.

**Pre-1.0 churn.** Living on `0.x` for years meant occasional breaking changes across minor bumps and API surface that moved between releases[^3]. Pin exact versions, read release notes before upgrading, and treat drizzle-orm and drizzle-kit as a version pair — mismatches between them cause confusing migration and snapshot errors.

**Multiple drivers per database is a footgun.** For Postgres alone there are `node-postgres`, `postgres-js`, `neon-http`, `neon-serverless`, `vercel-postgres`, and more; each has different connection, pooling, and edge-compatibility behavior. Choosing the wrong driver for your runtime (e.g. a TCP driver on an edge platform) fails at deploy time, not build time.

## When to Use / When Not

**Use when:**
- You want static type safety but refuse to lose sight of the SQL you emit.
- You target serverless/edge runtimes and need a zero-dependency library that wraps the driver you already use.
- You want reviewable, plain-SQL migrations checked into the repo.
- You are a toolmaker building on top of a typed but thin SQL layer.

**Avoid when:**
- You want maximum terseness and a batteries-included schema DSL — Prisma's higher-level model is less code for CRUD-heavy apps.
- Your team is uncomfortable writing SQL; Drizzle assumes SQL fluency by design.
- You need a stable, rarely-breaking API today and cannot absorb migration work across releases.
- Your schema is enormous and TypeScript build/editor performance is already a bottleneck.

## Alternatives

- prisma/prisma — use instead when you want a schema DSL, generated client, and broad tooling, and can accept the higher-level abstraction and query engine.
- kysely-org/kysely — use instead when you want a pure type-safe query builder with no ORM schema, relations, or migrations.
- typeorm/typeorm — use instead when you want decorator-based entities and Active Record/Data Mapper patterns from a mature (if heavier) codebase.
- knex/knex — use instead when you want a battle-tested, untyped JS query builder and migration tool without a TypeScript-first design.
- sequelize/sequelize — use instead for a long-established ORM with the widest legacy footprint, accepting weaker TypeScript ergonomics.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2021-06-24 | Initial development by Drizzle Team[^1]. |
| public releases | 2022 | SQL-like builder, Postgres/MySQL/SQLite dialects[^1]. |
| Drizzle Kit | 2022–2023 | SQL migration generation + `studio` browser[^4]. |
| Relational Query Builder | 2023 | Nested `db.query.*` reads via lateral joins[^5]. |
| RQB v2 redesign | 2024–2025 | Rework of the relational layer for query performance[^5]. |

(Drizzle published under `0.x` version numbers throughout this period[^3]; consult the repository releases for exact version-to-date mapping.)

## References

[^1]: Drizzle ORM README and documentation overview. https://orm.drizzle.team/docs/overview
[^2]: Bundle size (~7.4 kB min+gzip, 0 dependencies) — Bundlephobia. https://bundlephobia.com/package/drizzle-orm
[^3]: drizzle-orm releases (version history). https://github.com/drizzle-team/drizzle-orm/releases
[^4]: Drizzle Kit documentation — migrations and Studio. https://orm.drizzle.team/kit-docs/overview
[^5]: Drizzle Relational Query Builder documentation. https://orm.drizzle.team/docs/rqb

## Tags

typescript, orm, sql, postgres, mysql, sqlite, query-builder, database, serverless, edge, migrations, type-safety
