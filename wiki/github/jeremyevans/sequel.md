# jeremyevans/sequel

> A Ruby SQL toolkit and ORM whose defining bet is immutable, chainable datasets over convention-heavy magic.

[GitHub repo](https://github.com/jeremyevans/sequel) ·
[Official website](https://sequel.jeremyevans.net) ·
[License: MIT](https://github.com/jeremyevans/sequel/blob/master/MIT-LICENSE)

## Overview

Sequel is a database access library for Ruby that spans two layers: a low-level
SQL query and schema DSL built on immutable `Dataset` objects, and an optional
ActiveRecord-pattern ORM (`Sequel::Model`) layered on top of it[^1]. It was
originally written by Sharon Rosner; Jeremy Evans has been the maintainer since
2008 and drives essentially all development[^2]. At ~5.1k stars it is smaller
than Rails' ActiveRecord in mindshare but is the reference "other" Ruby ORM, and
the usual choice for teams that want SQL control without leaving Ruby.

The defining design tension is explicitness versus convenience. Where
ActiveRecord globally monkeypatches core classes and auto-loads most behavior,
Sequel keeps its surface minimal by default: datasets are frozen and return new
copies on every method call, the "core extensions" that add methods to `Symbol`
and `String` are opt-in, and nearly every capability (JSON columns, pagination,
soft-deletes, PostgreSQL types) lives in a separately-loaded extension or model
plugin. You assemble the ORM you want rather than inheriting one wholesale.

The other notable trait is the maintenance model. The issue tracker sits at or
near zero open issues by policy — bugs are triaged and closed quickly, and the
project ships small releases on a roughly monthly cadence with a documented,
strict semantic-versioning discipline and a maintained CHANGELOG[^3]. This is
unusually disciplined for an OSS library of this age (the repo dates to 2008).

## Getting Started

```bash
gem install sequel
# plus a driver gem for your database, e.g.:
gem install pg        # PostgreSQL
gem install sqlite3   # SQLite
```

```ruby
require 'sequel'

DB = Sequel.connect('postgres://user:pass@localhost/mydb')

# Dataset layer — no models needed
items = DB[:items]
items.insert(name: 'abc', price: 12.5)
items.where { price > 10 }.order(:name).map(:name)
# SELECT name FROM items WHERE (price > 10) ORDER BY name

# Model layer — active-record pattern over a table
class Item < Sequel::Model
  # columns are read from the DB schema at load time
end
Item.where(name: 'abc').first.update(price: 9.99)
```

## Architecture / How It Works

Sequel is layered deliberately, and the layers are usable in isolation.

- **Database** — a connection object plus a thread-safe connection pool. One
  `Sequel::Database` instance per database; the docs push the convention of
  storing it in a `DB` constant. Handles transactions, savepoints, sharding, and
  primary/replica routing.
- **Dataset** — an object encapsulating one SQL query. Datasets are **frozen and
  immutable**: `ds.where(...)` returns a new dataset rather than mutating the
  receiver, which makes them safe to share across threads and safe to store and
  reuse. SQL is generated lazily and only executed when you call a terminal
  method (`all`, `each`, `first`, `count`, `map`, ...).
- **Model** — `Sequel::Model` wraps a dataset; an instance wraps one row. It
  reads the table schema at class-definition time to define accessors. Behavior
  beyond CRUD comes from **plugins** (`plugin :timestamps`, `:validation_helpers`,
  `:single_table_inheritance`, `:json_serializer`, etc.).

The query DSL has two dialects. The plain-hash form (`where(category: 'ruby')`)
covers equality/IN/range cases; the **virtual-row block** form (`where { price >
10 }`) uses `method_missing` to build expression trees — the most "magical" part
of the API. `Sequel.lit` drops to raw SQL with safe placeholder interpolation.

Adapters are split into a shared adapter (SQL generation, per-database quirks)
and a driver adapter (the actual C/JDBC binding: `pg`, `mysql2`, `trilogy`,
`sqlite3`, `jdbc/*`, and others). This is why Sequel runs on MRI and JRuby with
the same code, and why database-specific features (PostgreSQL arrays, `hstore`,
`jsonb`, ranges, `LISTEN`/`NOTIFY`, MERGE, window functions, CTEs) are gated
behind explicit extension loads rather than always present.

## Production Notes

- **`exclude` inverts the whole filter.** `exclude(a: 1, b: 2)` produces
  `NOT (a = 1 AND b = 2)` → `a != 1 OR b != 2`, not two separate negations. The
  README calls this out explicitly; it is a recurring source of wrong result
  sets for people who expect AND-of-negations[^1].
- **Eager loading has two mechanisms and they are not interchangeable.**
  `eager` issues separate queries per association (avoids row multiplication);
  `eager_graph` does a single `LEFT JOIN`. Reaching for `eager_graph` on
  one-to-many associations can trigger a cartesian blow-up in rows fetched;
  `eager` avoids it but costs extra round-trips. Filtering on associated tables
  usually forces `eager_graph`.
- **Symbol splitting was removed as a default in Sequel 5.** Older code relied on
  `:table__column` / `:column___alias` magic embedded in symbols; that behavior
  is now opt-in via the `symbol_aliases`/`split_symbols` extensions. Code
  migrated from Sequel 4 that used the double/triple-underscore convention will
  silently query wrong identifiers until updated to `Sequel[:table][:column]`[^4].
- **Frozen datasets (default since 5.0).** Any code from the 3.x/4.x era that
  mutated a dataset in place will raise. This was the headline breaking change of
  the 5.0 line and is the main reason large upgrades are non-trivial[^4].
- **Model schema is read at boot.** Because `Sequel::Model` introspects the table
  at class-load time, the database must be reachable when your models load. This
  affects boot ordering, CI, and asset-precompile-style steps that don't have a
  DB. `Model.freeze` / setting columns explicitly are the escape hatches.
- **Security is your responsibility at the `lit` boundary.** The DSL parameterizes
  by default, but `Sequel.lit` with string interpolation reintroduces injection
  risk. The project ships a dedicated security guide[^5]. Migrations also come in
  two flavors (integer vs. timestamped) that should not be mixed in one app.

## When to Use / When Not

**Use when:**
- You want fine SQL control (CTEs, window functions, MERGE, PG-specific types)
  without hand-writing strings.
- You value immutability/thread-safety and want to reuse composed queries.
- You want the dataset layer without an ORM, or an ORM you assemble from plugins.
- You run on JRuby, use sharding/replicas, or need database features Rails
  abstracts away.

**Avoid when:**
- You're deep in Rails and want the default, ecosystem-blessed path — ActiveRecord
  has far more third-party integration and Stack Overflow surface area.
- Your team expects convention-over-configuration and dislikes opt-in wiring.
- You need a large hiring pool already fluent in the ORM; Sequel expertise is
  rarer than ActiveRecord.

## Alternatives

- rails/rails — ActiveRecord; use instead when you're already in Rails and want
  the batteries-included default with the widest ecosystem.
- rom-rb/rom — data-mapper (not active-record); use when you want domain objects
  fully decoupled from persistence and explicit repositories.
- hanami/hanami — full-stack Ruby framework whose persistence sits on ROM; use
  when you want that architecture end to end.
- ged/ruby-pg — the raw PostgreSQL driver; use when you want zero abstraction and
  will write and manage all SQL yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2007 | Originally created by Sharon Rosner[^2]. |
| — | 2008 | Jeremy Evans becomes maintainer; repo opened on GitHub[^2]. |
| 3.0 | 2009 | Major API consolidation of the dataset/model split. |
| 4.0 | 2013 | Large cleanup release; deprecations from the 3.x line removed. |
| 5.0 | 2017 | Frozen datasets by default; symbol-splitting magic made opt-in[^4]. |
| 5.x | 2017–present | Roughly monthly point releases under strict semver[^3]. |

## References

[^1]: Sequel README and introduction (dataset DSL, `exclude` inversion, model layer). https://github.com/jeremyevans/sequel/blob/master/README.rdoc
[^2]: Sequel project history and maintainership. https://sequel.jeremyevans.net/
[^3]: Sequel CHANGELOG (release cadence and semver discipline). https://github.com/jeremyevans/sequel/blob/master/CHANGELOG
[^4]: "Sequel 5 upgrade guide" — frozen datasets and removed symbol splitting. https://sequel.jeremyevans.net/rdoc/files/doc/migrating_to_sequel_5_rdoc.html
[^5]: Sequel Security Guide. https://sequel.jeremyevans.net/rdoc/files/doc/security_rdoc.html

## Tags

ruby, orm, sql, database, postgresql, mysql, sqlite, query-builder, activerecord-alternative, data-mapper, jruby
