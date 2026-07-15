# mikro-orm/mikro-orm

> TypeScript ORM for Node.js built on the Data Mapper, Unit of Work, and Identity Map patterns — the Doctrine/Hibernate model ported to the JS ecosystem.

[GitHub repo](https://github.com/mikro-orm/mikro-orm) ·
[Official website](https://mikro-orm.io) ·
[License: MIT](https://github.com/mikro-orm/mikro-orm/blob/master/LICENSE)

## Overview

MikroORM is a TypeScript ORM for Node.js authored by Martin Adámek (`@b4nan`), first published in 2018[^1]. Its explicit reference points are Doctrine (PHP) and Hibernate (Java): it implements the **Data Mapper** pattern, an **Identity Map**, and a **Unit of Work** rather than the Active Record style common to Sequelize and TypeORM's default usage. Entities are plain classes with no base class and no persistence methods; a separate `EntityManager` owns loading, change tracking, and flushing.

The defining mechanism is the Unit of Work. Loaded entities are held in an identity map (one in-memory instance per primary key per EntityManager). You mutate entity objects directly, and on `em.flush()` MikroORM diffs the tracked entities against their original snapshots, computes a minimal set of change sets, orders them by dependency, and executes them inside a single transaction. There is no per-object `.save()`. This buys automatic dirty checking and referential consistency, at the cost of a mental model that surprises developers coming from query-builder or Active Record tools.

That same identity map is the framework's central tension. A single shared EntityManager will accumulate entities and leak them across unrelated requests, so server applications must give each request its own forked context (`RequestContext` or `em.fork()`). Getting this wrong is the most common production incident in MikroORM codebases, and it is the price of the Doctrine-style design the project deliberately chose. MikroORM is unusual in also covering MongoDB under the same Data Mapper API, alongside the SQL drivers[^2].

## Getting Started

Install the driver package for your database (the core is pulled in transitively):

```sh
npm install @mikro-orm/postgresql   # or mysql, mariadb, sqlite, libsql,
                                    # mssql, oracledb, mongodb, pglite
```

```typescript
import { Entity, PrimaryKey, Property, MikroORM } from '@mikro-orm/postgresql';

@Entity()
class User {
  @PrimaryKey()
  id!: number;

  @Property()
  name!: string;

  @Property({ unique: true })
  email!: string;
}

const orm = await MikroORM.init({
  entities: [User],
  dbName: 'my_db',
});

// fork() gives this unit of work its own identity map
const em = orm.em.fork();
em.create(User, { name: 'Jon Snow', email: 'snow@wall.st' });

// change set computed and committed in a single transaction
await em.flush();
```

Entities can also be declared with the schema-first `defineEntity` API or the lower-level `EntitySchema`, both of which avoid decorators entirely[^3].

## Architecture / How It Works

**Metadata discovery** is where much of MikroORM's complexity lives. The ORM needs the shape and relations of every entity. It obtains this in one of three ways:

- `ReflectMetadataProvider` — reads `reflect-metadata` emitted by `tsc` with `emitDecoratorMetadata`. Requires every property to declare an explicit `type` in its decorator, because reflected metadata collapses to `Object` for anything non-primitive.
- `TsMorphMetadataProvider` (`@mikro-orm/reflection`) — parses the actual TypeScript source with ts-morph to infer types. More ergonomic, but needs the `.ts` files (or a generated metadata cache) available and is sensitive to build/bundler setups.
- `defineEntity` / `EntitySchema` — types are supplied explicitly in code, so no reflection is needed at all.

**The Unit of Work** maintains original snapshots of managed entities. `flush()` runs change-set computation (insert/update/delete), topologically orders them to respect foreign keys, batches same-type operations (e.g. a single multi-row `update ... case when`), and wraps everything in one transaction. Cascading, orphan removal, and collection synchronization all resolve here.

**SQL drivers** are built on top of Knex.js, which handles connection pooling, dialect-specific SQL generation, and the low-level query execution; MikroORM's `QueryBuilder` sits above it and returns managed entities that re-enter the identity map. The MongoDB driver is a separate implementation over the official Node driver.

**Loading strategies** control how relations are fetched: `select-in` (a follow-up `IN (...)` query per relation, the default) versus `joined` (a single query with joins, deserialized back into the object graph). The choice materially affects query count and row duplication.

Supporting subsystems — migrations (via `umzug`), the schema generator (diff-based DDL), the entity generator (reverse-engineering a schema into entities), and the CLI — ship as separate `@mikro-orm/*` packages, so applications only pull in what they use.

## Production Notes

**Request isolation is mandatory.** Sharing `orm.em` directly across HTTP requests leaks the identity map and produces cross-request data bleed and unbounded memory growth. Wrap request handling in `RequestContext.create(orm.em, next)` or fork per request. This is the single most frequent MikroORM footgun.

**Bundlers strip the metadata MikroORM relies on.** esbuild, swc, and by extension many serverless/Next.js/Vite setups do not emit `reflect-metadata` and do not preserve source for ts-morph. In these environments you typically must pre-generate a metadata cache (`mikro-orm cache:generate --combined`) at build time or move to `defineEntity`/`EntitySchema`, which need no reflection. Expect friction on any bundled or serverless deployment.

**`flush()` is all-or-nothing per unit of work.** Forgetting to flush silently drops changes; flushing computes a diff over every managed entity, so very large identity maps make flush expensive. Long-lived EntityManagers that accumulate thousands of entities should be forked or cleared (`em.clear()`).

**QueryBuilder bypasses parts of the abstraction.** Raw `qb.execute()` returns plain rows, not managed entities; use `getResult()`/`getResultList()` to get objects that participate in the identity map. Mixing the two is a common source of "why aren't my changes persisting" confusion.

**Serialization can over-fetch.** Because relations are lazy proxies, naively `JSON.stringify`-ing an entity can trigger loads or expose more of the graph than intended. Use explicit `serialize()` / DTOs and `populate` hints to bound what is loaded and returned.

**Upgrades are periodic and non-trivial.** Major versions (v4, v5, v6) each carried breaking changes to configuration, driver package layout, and entity APIs; migrations between them require reading the dedicated upgrade guides rather than a mechanical bump[^2].

## When to Use / When Not

**Use when:**
- You want true Data Mapper / Unit of Work semantics (automatic dirty checking, one transactional flush) rather than manual `.save()` calls.
- Your team already thinks in Doctrine/Hibernate terms and wants that model in Node.
- You need one entity API spanning several SQL databases and MongoDB.
- You value strong TypeScript inference on `populate`, relations, and query results.

**Avoid when:**
- You are deploying to a bundled/serverless target and don't want to manage metadata caching or move off decorators.
- You prefer a thin, explicit SQL layer with no hidden change tracking — the identity map will feel like magic working against you.
- The team is small and the Unit of Work learning curve outweighs its benefit for a simple CRUD app.
- You want the largest community and answer base; TypeORM and Prisma have more surface coverage.

## Alternatives

- prisma/prisma — schema-first client generator with excellent DX; use instead when you want type-safe queries without an identity map or Unit of Work, and accept a code-generation step.
- typeorm/typeorm — supports both Active Record and Data Mapper with a larger user base; use when ecosystem size matters more than the cleaner internals, despite its heavier legacy baggage.
- drizzle-team/drizzle-orm — thin TypeScript SQL query builder with no change tracking; use when you want SQL-level control and minimal abstraction.
- sequelize/sequelize — mature JS-first Active Record ORM; use for older JavaScript codebases or when you don't need strict typing.
- kysely — type-safe query builder, not an ORM; use when you want compile-time-checked SQL without entities, identity maps, or migrations built in.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019 | Early stable line; decorator entities, SQL + MongoDB drivers. |
| 4.0 | 2020-10 | Package split into `@mikro-orm/*` drivers, config overhaul[^2]. |
| 5.0 | 2022-02 | Reworked collections, optimistic locking, API cleanups[^2]. |
| 6.0 | 2024-01 | Type-safety overhaul, `defineEntity`, stricter partial loading[^2]. |

## References

[^1]: MikroORM repository and author (Martin Adámek, `@b4nan`). https://github.com/mikro-orm/mikro-orm
[^2]: MikroORM releases and upgrade guides. https://github.com/mikro-orm/mikro-orm/releases
[^3]: MikroORM docs, "Defining Entities" (decorators, `EntitySchema`, `defineEntity`). https://mikro-orm.io/docs/defining-entities

## Tags

typescript, orm, nodejs, data-mapper, unit-of-work, identity-map, postgresql, mongodb, sql, database, active-record-alternative, knex
