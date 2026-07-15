# sequelize/sequelize

> A mature, active-record-style ORM for Node.js that trades type-safety for broad SQL-dialect coverage.

[GitHub repo](https://github.com/sequelize/sequelize) ·
[Official website](https://sequelize.org) ·
[License: MIT](https://github.com/sequelize/sequelize/blob/main/LICENSE)

## Overview

Sequelize is one of the oldest surviving Node.js ORMs — the repository was created
in 2010[^1] — and for most of the last decade it was the default answer to "how do
I talk to a SQL database from Node." It follows the active-record pattern: you define
model classes, and instances of those classes carry both data and persistence methods
(`user.save()`, `user.reload()`, `user.destroy()`). It targets an unusually wide set
of backends: PostgreSQL, MySQL, MariaDB, SQLite, Microsoft SQL Server, Snowflake,
Oracle Database, DB2, and DB2 for IBM i[^2]. That dialect breadth is its main reason
to exist and the main thing newer ORMs do not match.

The defining tension is age. Sequelize predates TypeScript's rise, `async/await`, and
the current generation of type-first ORMs, and it shows. The stable line (version 6,
published on npm as `sequelize`) is a JavaScript library with TypeScript typings bolted
on afterward; getting good inference requires ceremony (`InferAttributes`,
`InferCreationAttributes`, `declare` fields) and still leaks `any` at the edges. The
next-generation rewrite (version 7, published as `@sequelize/core`) reworks this with
decorator-based models and per-dialect packages, but has been in alpha for a long
stretch[^3]. In 2026 the project is actively maintained but visibly under-resourced:
the README openly solicits new maintainers and funds the effort through OpenCollective
at roughly $2,500 per quarter[^4].

## Getting Started

```bash
# Stable line (v6)
npm install sequelize
npm install pg pg-hstore   # driver for your dialect (pg, mysql2, sqlite3, tedious, ...)
```

```js
const { Sequelize, DataTypes, Model } = require("sequelize");

const sequelize = new Sequelize("postgres://user:pass@localhost:5432/mydb");

class User extends Model {}
User.init(
  {
    id:    { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    email: { type: DataTypes.STRING, allowNull: false, unique: true },
  },
  { sequelize, modelName: "user" },
);

await sequelize.authenticate();          // verify the connection
await User.sync();                        // CREATE TABLE IF NOT EXISTS (dev only)
const u = await User.create({ email: "a@example.com" });
const found = await User.findOne({ where: { email: "a@example.com" } });
```

## Architecture / How It Works

A `Sequelize` instance owns a **connection pool** (backed by `sequelize-pool`) and a
**dialect abstraction**. Every query is assembled by a dialect-specific query generator
that emits SQL strings, then run through the matching driver (`pg`, `mysql2`,
`sqlite3`, `tedious`, etc.). The driver is a peer dependency you install yourself —
Sequelize ships no bundled database client.

Models are classes extending `Model`. In v6 you register fields via `Model.init(...)`
or the `sequelize.define(...)` shorthand; in v7 you annotate class properties with
decorators (`@Attribute`, `@NotNull`, `@BelongsTo`). Associations
(`hasMany`, `belongsTo`, `belongsToMany`, `hasOne`) are declared imperatively and
generate helper methods, foreign-key wiring, and the join logic used by eager loading.

Eager loading is the part worth understanding before you commit. `include` pulls
associated rows, and by default Sequelize renders them as SQL `JOIN`s that return a
denormalized cartesian result, which the ORM then re-nests into object graphs in
JavaScript. Deeply nested includes combined with `limit` force Sequelize into a
subquery strategy that surprises people, and `hasMany` includes can be split into
separate queries with `separate: true` to avoid row multiplication. Migrations are a
distinct concern handled by the separate `sequelize-cli`[^5]; the model layer and the
migration layer do not share a source of truth, so schema and models can silently drift.

## Production Notes

**Never rely on `sync()` in production.** `Model.sync()` and `sequelize.sync({ alter: true })`
issue DDL derived from your model definitions. `alter` in particular can drop or rewrite
columns in ways that lose data, and it does not produce a reviewable migration. Production
schema changes belong in versioned migrations run by `sequelize-cli`, kept separate from
model definitions.

**Eager-loading footguns.** The most common performance complaints are (1) N+1 queries
from lazy-loading associations inside a loop, and (2) enormous cartesian result sets from
nested `include`s. `include` + `limit` behaves differently from what many expect because
of the subquery rewrite; when you need the top N parents with all their children, reach
for `separate: true` or split the query yourself.

**TypeScript is workable but not first-class in v6.** You must declare model attributes
twice in effect — once for the runtime via `init`, once for the compiler via
`InferAttributes`/`InferCreationAttributes` and `declare` fields — and it still cannot
type raw queries or many `where` operators precisely. If end-to-end type safety is a hard
requirement, this is where teams leave for Prisma, Drizzle, or MikroORM.

**Raw queries: use `bind`, not string interpolation.** `sequelize.query` supports
`replacements` (client-side substitution) and `bind` (real parameterized placeholders).
Prefer `bind` for user input; `replacements` is escaping, not binding.

**Version choice.** v6 is the version you should run in production today. v7 (`@sequelize/core`)
is cleaner but has spent a long time in alpha, splits each dialect into its own package,
and carries breaking API changes — treat it as forward-looking, not a drop-in upgrade[^3].
The v5→v6 and v6→v7 jumps both have dedicated upgrade guides for a reason: they are not
mechanical.

## When to Use / When Not

**Use when:**
- You need a dialect Sequelize supports but Prisma/Drizzle do not (Oracle, DB2, Snowflake, MSSQL, IBM i).
- You want the active-record style — data and behavior on the same object, hooks, instance validation.
- You have an existing Sequelize codebase; it is stable and well-trodden, with vast Stack Overflow coverage.
- You prefer a runtime-JS library over a codegen/schema-file workflow.

**Avoid when:**
- Compile-time type safety across queries is a primary requirement — v6's typings are add-on and leaky.
- You want generated migrations tied to your schema of record — Sequelize keeps models and migrations separate.
- You are starting greenfield on Postgres/MySQL only and value a modern DX — Drizzle or Prisma fit better.
- You need the newest ORM ergonomics and can wait — v7's improvements are real but still alpha.

## Alternatives

- prisma/prisma — schema-file + codegen ORM with strong types and generated migrations; use instead when type safety and DX matter more than dialect breadth.
- drizzle-team/drizzle-orm — thin, SQL-shaped, fully-typed query builder; use when you want types without a heavy runtime or codegen step.
- typeorm/typeorm — decorator-based ORM supporting both active-record and data-mapper; use when you want a similar feature surface with a decorator-first API.
- mikro-orm/mikro-orm — TypeScript-first data-mapper with a unit-of-work/identity-map model; use when you want proper unit-of-work semantics and strong typing.
- knex/knex — query builder, not an ORM (and what several ORMs build on); use when you want to write SQL-shaped queries without model abstractions.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2010-07 | Repository created; among the earliest Node.js ORMs[^1]. |
| 4.x | ~2017 | Dropped the custom promise library; native promises and async/await. |
| 5.x | ~2019 | Stricter defaults, dialect and data-type cleanup. |
| 6.x | ~2020 | Current stable line; improved (but add-on) TypeScript typings, `InferAttributes` later in the series. |
| 7.x (alpha) | ongoing | Rewrite as `@sequelize/core`; decorator models, per-dialect packages, breaking API changes[^3]. |

## References

[^1]: Repository metadata (`created_at` 2010-07-22), GitHub API — repos/sequelize/sequelize.
[^2]: Repository description and dialect list. https://github.com/sequelize/sequelize
[^3]: Sequelize v7 getting-started and v6→v7 upgrade guide. https://sequelize.org/docs/v7/getting-started/
[^4]: Sequelize README, "Seeking New Maintainers" and OpenCollective funding. https://opencollective.com/sequelize
[^5]: sequelize-cli — migrations and seeders. https://github.com/sequelize/cli

## Tags

typescript, javascript, nodejs, orm, sql, postgresql, mysql, active-record, database, migrations
