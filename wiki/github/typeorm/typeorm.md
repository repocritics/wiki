# typeorm/typeorm

> TypeScript/JavaScript ORM for Node.js that supports both the Active Record and Data Mapper patterns across a large set of SQL databases.

[GitHub repo](https://github.com/typeorm/typeorm) ·
[Official website](http://typeorm.io) ·
[License: MIT](https://github.com/typeorm/typeorm/blob/master/LICENSE)

## Overview

TypeORM is a decorator-driven ORM for Node.js, originally created by Umed Khudoiberdiev (pleerock) in 2016[^1]. It models tables as TypeScript classes annotated with `@Entity`, `@Column`, and relation decorators, and it deliberately supports two competing usage styles: Data Mapper (repositories separate from entities) and Active Record (entities extend `BaseEntity` and carry their own persistence methods). It targets an unusually wide database matrix — Postgres, MySQL/MariaDB, SQLite, SQL Server, Oracle, CockroachDB, SAP HANA, Google Spanner, and MongoDB — which is its headline differentiator over most JS ORMs[^2].

The defining tension is between breadth and reliability. TypeORM covers more databases and more ORM features (closure-table trees, multiple inheritance strategies, replication, query caching, subscribers, migrations) than any competitor, but that surface area has historically outpaced its maintenance capacity. The project spent long stretches with a large open-issue backlog and slow release cadence, and it is still on a `0.x` version after roughly nine years — a fact that matters for teams reading semver literally.

It remains one of the most-installed TS ORMs and the default in a large chunk of the NestJS ecosystem, but since ~2022 it has been losing greenfield mindshare to Prisma and Drizzle, which offer stronger type-safety guarantees and clearer maintenance signals.

## Getting Started

```bash
npm install typeorm reflect-metadata
npm install pg   # or mysql2, better-sqlite3, mssql, etc.
```

TypeORM relies on legacy (stage-2) decorators and requires `reflect-metadata` plus `experimentalDecorators` and `emitDecoratorMetadata` in `tsconfig.json`[^3]:

```typescript
import "reflect-metadata"
import { DataSource, Entity, PrimaryGeneratedColumn, Column } from "typeorm"

@Entity()
export class User {
    @PrimaryGeneratedColumn() id: number
    @Column() firstName: string
    @Column() age: number
}

const AppDataSource = new DataSource({
    type: "postgres",
    url: process.env.DATABASE_URL,
    entities: [User],
    synchronize: false,   // never true in production — see Production Notes
})

await AppDataSource.initialize()
const repo = AppDataSource.getRepository(User)
await repo.save(repo.create({ firstName: "Timber", age: 25 }))
const adults = await repo.findBy({ age: 25 })
```

## Architecture / How It Works

At startup TypeORM reads decorator metadata (via `reflect-metadata`) into an internal **entity metadata** graph, then builds a `DataSource` (before 0.3.0 this was called `Connection`)[^4]. The `DataSource` owns the driver, the connection pool, and the `EntityManager`; repositories are thin per-entity views over the `EntityManager`.

Two query surfaces coexist and behave differently:

1. **Find API** (`find`, `findOne`, `findBy`) — declarative option objects. Relations requested here are resolved either by `LEFT JOIN` (default) or by separate queries (`relationLoadStrategy: "query"`).
2. **QueryBuilder** — a fluent SQL builder for anything the Find API cannot express (subqueries, complex joins, raw expressions, locking). It is where most non-trivial production queries end up, and it is closer to writing SQL than to using an abstraction.

Relations are declared with `@OneToMany` / `@ManyToOne` / `@ManyToMany` / `@OneToOne`. They can be **eager** (always loaded), **lazy** (typed as `Promise<T>`, loaded on access), or explicit. **Migrations** are TypeScript classes with `up`/`down`; `migration:generate` diffs the current schema against entity metadata and emits DDL. **Subscribers** and entity listeners provide lifecycle hooks (`@BeforeInsert`, etc.).

The decorator-metadata design is the root of both its ergonomics and its fragility: schema truth lives in TypeScript types plus runtime reflection, so anything that breaks metadata emission (SWC/esbuild without the decorator-metadata plugin, TC39 standard decorators, bundler tree-shaking) breaks the ORM in ways that surface only at runtime.

## Production Notes

**`synchronize: true` is a data-loss footgun.** It auto-alters the schema to match entities on every boot and will silently drop columns/tables. It is convenient in dev and catastrophic in production; use migrations instead[^5].

**Eager relations and N+1.** Relations marked `eager: true` load on every `find`, which quietly over-fetches. Even non-eager relations loaded via the default join strategy can multiply rows and **break `take`/`skip` pagination**, because `LIMIT` applies to the joined row set. The historical fix is `relationLoadStrategy: "query"` (separate queries per relation), added to address exactly this class of bug; older code predating it often paginates incorrectly.

**Migration generation is a starting point, not an oracle.** `migration:generate` produces noisy or occasionally incorrect diffs, especially across enum changes, default values, and non-Postgres dialects. Generated migrations should always be read and edited by hand before shipping.

**Two ways to do everything.** Active Record vs Data Mapper, Find API vs QueryBuilder, `save()` (upsert-ish, runs a SELECT then INSERT/UPDATE) vs `insert()`/`update()` (direct, no cascades or listeners). `save()`'s implicit read-before-write surprises people under load.

**Lazy relations change types.** Switching a relation to lazy changes its type from `T` to `Promise<T>`, a source-level breaking change that ripples through call sites.

**MongoDB support is second-class.** The Mongo driver reuses the SQL-oriented API imperfectly; teams doing serious document work usually reach for a native driver or Mongoose instead.

**Maintenance signal.** The issue tracker has historically carried a large backlog and the release cadence has been uneven. This is improving but is worth weighing for a dependency you expect to hold schema-critical code for years.

## When to Use / When Not

**Use when:**
- You need a single ORM spanning several SQL engines (or an unusual one like SAP HANA, Spanner, Oracle).
- You're in NestJS, where TypeORM is a first-class, well-trodden integration.
- You want mature migrations, subscribers, tree structures, and replication in one package.
- You prefer decorator-on-class modeling and are comfortable dropping to QueryBuilder for hard queries.

**Avoid when:**
- You want end-to-end static type safety on query results — Prisma and Drizzle infer far more precisely.
- Your build uses SWC/esbuild/bundlers where decorator metadata is awkward, or you want to avoid `reflect-metadata` entirely.
- You value a stable, actively-triaged dependency with a 1.0 guarantee and predictable releases.
- Your workload is primarily MongoDB/document-oriented.

## Alternatives

- prisma/prisma — schema-file-driven with a generated, strongly-typed client; use when you want the best result-type inference and a managed migration workflow, and can accept its engine/codegen model.
- drizzle-team/drizzle-orm — SQL-first, no decorators, lightweight and fully typed; use when you want thin abstraction close to SQL with minimal runtime.
- mikro-orm/mikro-orm — also decorator/Data-Mapper based but with a real Identity Map and Unit of Work; use when you want TypeORM-style modeling with stronger persistence semantics.
- sequelize/sequelize — older, JS-first, huge install base; use for legacy Node codebases already standardized on it.
- kysely-org/kysely — a typed query builder, not an ORM; use when you want SQL and types but no entity/relation layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-02 | First commits as a TypeScript ORM by pleerock[^1]. |
| 0.1.0 | 2017-09 | First 0.1 line; decorator entities, repositories, QueryBuilder. |
| 0.2.0 | 2018-04 | Long-lived stable line; broad driver support matured[^6]. |
| 0.3.0 | 2022-03 | `Connection` → `DataSource`; global `getRepository`/`getManager` helpers deprecated; ESM support work[^4]. |
| 0.3.x | 2022–2026 | Ongoing 0.3 releases; `relationLoadStrategy`, driver fixes, TS support updates. |

## References

[^1]: TypeORM repository, created 2016-02-29. https://github.com/typeorm/typeorm
[^2]: TypeORM README — supported databases and Active Record / Data Mapper patterns. https://github.com/typeorm/typeorm/blob/master/README.md
[^3]: TypeORM installation and decorator/`reflect-metadata` requirements. https://typeorm.io/#installation
[^4]: TypeORM 0.3.0 release notes — DataSource API and deprecations. https://github.com/typeorm/typeorm/releases/tag/0.3.0
[^5]: TypeORM DataSource options — `synchronize` warning. https://typeorm.io/data-source-options
[^6]: TypeORM releases. https://github.com/typeorm/typeorm/releases

## Tags

typescript, javascript, orm, database, sql, postgresql, mysql, active-record, data-mapper, migrations, nodejs, decorators
