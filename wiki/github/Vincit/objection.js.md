# Vincit/objection.js

> An SQL-friendly ORM for Node.js that treats queries — not entities — as the primary abstraction, built as a relation-aware layer on top of the knex query builder.

[GitHub repo](https://github.com/Vincit/objection.js) ·
[Official website](https://vincit.github.io/objection.js) ·
[License: MIT](https://github.com/Vincit/objection.js/blob/main/LICENSE)

## Overview

objection.js is a data-mapping library for Node.js authored at Vincit, a Finnish
software consultancy, with first commits in 2015[^1]. Its own README argues that
"ORM" is a misleading label and prefers "relational query builder": you do not
manipulate persistent entities that transparently sync to the database, you
build and execute queries whose results happen to be model instances. This is
the defining design stance — objection deliberately does *not* give you a fully
object-oriented view of the database, a custom query DSL, or schema
generation/migration from model definitions[^2].

Everything is layered on knex[^3]. Models declare a `tableName`, an optional
`jsonSchema` for validation, and `relationMappings` describing joins; queries
start from `Model.query()` and expose the full knex builder plus objection's
relation operators. Because knex is the SQL engine, every database knex supports
works, with PostgreSQL, MySQL, and SQLite3 being the thoroughly tested set[^2].
The trade this buys: raw SQL is always one `.raw()` away and there is little
magic between your code and the emitted query — at the cost of no automatic
migrations, no generated types beyond the hand-maintained TypeScript typings, and
no ActiveRecord-style change tracking.

The most important context for adopters in 2026 is maintenance posture. The
original author, Sami Koskimäki (koskimas), went on to build Kysely, a
type-safe SQL query builder, and objection now moves at a slow, mostly
bug-fix cadence rather than active feature development[^4]. It is stable and
widely deployed, but new greenfield projects should weigh that trajectory.

## Getting Started

```bash
npm install objection knex
# plus one driver:
npm install pg          # or: mysql2 / better-sqlite3 / sqlite3
```

```js
const { Model } = require('objection');
const Knex = require('knex');

// knex owns the connection pool and dialect
Model.knex(Knex({ client: 'pg', connection: process.env.DATABASE_URL }));

class Person extends Model {
  static get tableName() { return 'persons'; }
  static get relationMappings() {
    return {
      pets: {
        relation: Model.HasManyRelation,
        modelClass: Animal,
        join: { from: 'persons.id', to: 'animals.ownerId' },
      },
    };
  }
}

// A query is knex + relations. Nothing runs until awaited.
const adults = await Person.query()
  .where('age', '>=', 18)
  .withGraphFetched('pets')
  .orderBy('lastName');
```

Schema creation and migrations are intentionally out of scope — use knex's
migration CLI (`knex migrate:make` / `migrate:latest`) as a separate concern[^2].

## Architecture / How It Works

A `Model` subclass is metadata: static getters (`tableName`, `idColumn`,
`jsonSchema`, `relationMappings`) tell objection how to map rows and joins.
`Model.query()` returns a `QueryBuilder` that wraps a knex builder — chaining
`.where()`, `.select()`, `.join()` delegates straight to knex, while
objection-specific operators (`withGraphFetched`, `insertGraph`, `upsertGraph`,
`$relatedQuery`) sit on top.

Relation loading has two distinct engines with different failure modes:

- **`withGraphFetched`** issues a separate query per relation level and stitches
  results in JavaScript. It avoids row multiplication but produces multiple
  round-trips.
- **`withGraphJoined`** emits a single query with `LEFT JOIN`s. It is one
  round-trip but can trigger cartesian-style row explosion across
  many-to-many/nested relations, and requires careful column aliasing.

Choosing between them is a real operational decision, not an implementation
detail — the wrong one is a common performance surprise.

Graph writes (`insertGraph`, `upsertGraph`) accept a nested object/array and
resolve inserts, updates, relates, and unrelates across the tree in dependency
order within a transaction. This is objection's most differentiated feature and
also its sharpest edge: without an `allowGraph` allow-list, a graph write is
effectively mass-assignment over relations.

Validation is optional and runs on write: if a model exposes a `jsonSchema`,
objection validates input against it using ajv before insert/update. Lifecycle
hooks (`$beforeInsert`, `$afterFind`, `$beforeUpdate`, etc.) and query-level
`context`/`modifiers` provide the extension points. TypeScript support is
official but comes from hand-written declaration files rather than inference from
your schema — types describe the API surface, they do not derive column types
from the database.

## Production Notes

- **knex is a peer dependency and the versions are coupled.** Objection pins a
  compatible knex range; upgrading one without checking the other is a frequent
  break. Connection pooling, timeouts, and dialect quirks are all knex/tarn
  concerns, not objection's — tune them there.
- **`withGraphJoined` row explosion.** Deep or many-to-many graphs fetched via
  the joined strategy can return an order of magnitude more rows than expected.
  Default to `withGraphFetched` unless you have measured the join to be faster.
- **`upsertGraph` needs guardrails.** Always pair it with `allowGraph()` and
  explicit options (`insertMissing`, `noDelete`, `relate`, `unrelate`).
  Unconstrained, it will happily delete or re-relate rows a client did not intend
  to touch. It also does not batch as aggressively as hand-written SQL for large
  trees.
- **Transactions.** `Model.transaction()` and passing a `trx` through
  `.query(trx)` are explicit; a query bound to the default `Model.knex()` will
  *not* auto-enlist in an ambient transaction. Forgetting to thread `trx` through
  nested calls is a classic source of writes that escape the transaction.
- **Migrations are your problem.** There is no drift detection between
  `jsonSchema`, `relationMappings`, and the actual database. The three can
  silently diverge.
- **Maintenance velocity.** Issues and PRs are triaged slowly and features are
  largely frozen[^4]. For a stable app this is fine; for one expecting active
  upstream evolution or first-class type inference, it is a real constraint.

## When to Use / When Not

**Use when:**
- You want to write mostly SQL-shaped queries but need first-class relation
  eager-loading, graph inserts/upserts, and transactions.
- You already use knex and want a model/relation layer without hiding the SQL.
- You value a thin, predictable mapping over generated migrations and codegen.
- You are on PostgreSQL/MySQL/SQLite and want a mature, battle-tested library.

**Avoid when:**
- You want end-to-end type safety inferred from your schema (reach for a
  TypeScript-first tool instead).
- You want the ORM to own migrations and generate types/clients for you.
- You need an actively evolving project with a fast release cadence.
- You prefer decorator/entity ActiveRecord or DataMapper ergonomics.

## Alternatives

- kysely-org/kysely — type-safe query builder by objection's original author; use instead when you want inferred TypeScript types and no model/ORM layer.
- prisma/prisma — schema-first ORM with generated client and managed migrations; use instead when you want codegen and migrations owned by the tool.
- drizzle-team/drizzle-orm — TypeScript-first SQL-like ORM; use instead when you want lightweight typed queries close to SQL.
- typeorm/typeorm — decorator-based ORM; use instead when you want ActiveRecord/DataMapper entity patterns.
- knex/knex — the underlying query builder; use instead when you need no relations or model layer at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015–2018 | Long pre-1.0 period; core model/relation/graph API took shape. |
| 1.0 | 2019 | First stable major; `eager`/graph API consolidated[^5]. |
| 2.0 | 2020 | `withGraphFetched`/`withGraphJoined` naming, `allowGraph`; dropped older Node versions[^5]. |
| 3.0 | 2022 | knex peer-dep bump, dropped end-of-life Node releases, cleanup of deprecated APIs[^5]. |

## References

[^1]: Vincit/objection.js repository metadata (created 2015-04-14; MIT; ~7.3k stars, ~640 forks as of 2026-07). https://github.com/Vincit/objection.js
[^2]: objection.js README — "What objection.js gives you / doesn't give you"; knex-based, no schema generation, knex migrations recommended. https://github.com/Vincit/objection.js/blob/main/README.md
[^3]: knex.js — SQL query builder objection is built on. https://knexjs.org
[^4]: Kysely — type-safe SQL query builder by objection's original author (koskimas), commonly cited as the maintenance successor. https://kysely.dev
[^5]: objection.js changelog and v1→v2→v3 migration guide. https://vincit.github.io/objection.js/release-notes/migration.html

## Tags

javascript, typescript, nodejs, orm, sql, query-builder, knex, postgresql, mysql, sqlite, database
