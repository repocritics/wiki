# seancorfield/next-jdbc

> The de facto standard Clojure wrapper for JDBC — a deliberately low-level,
> protocol-based successor to `clojure.java.jdbc`.

[GitHub repo](https://github.com/seancorfield/next-jdbc) ·
[Official docs](https://cljdoc.org/d/com.github.seancorfield/next.jdbc/) ·
License: EPL-2.0

## Overview

`next.jdbc` (repo name `next-jdbc`) is Sean Corfield's ground-up rewrite of
`clojure.java.jdbc`, the Clojure contrib library he previously maintained. It
reached 1.0.0 in June 2019[^1] and has since displaced its predecessor as the
default way Clojure programs talk to relational databases over JDBC. The
design goals were explicit: less overhead when realizing `ResultSet`s (via
`IReduceInit`/transducers), a smaller and more consistent API surface, and
qualified keywords (`:table/column`) as the default row representation[^1].

The defining tension is that `next.jdbc` is *low-level on purpose*. It hands
you raw `javax.sql.DataSource` and `java.sql.Connection` objects rather than
wrapping them, does no SQL generation beyond a thin "friendly functions"
namespace, and pushes decisions like connection pooling and streaming
configuration onto you. That makes it predictable and fast, but it means most
real applications pair it with a SQL builder (HoneySQL) and a pool (HikariCP)
rather than using it alone.

At 861 stars it looks small next to mainstream-language libraries, but star
counts undercount Clojure infrastructure: this is the community-consensus
choice, and maintenance is unusually healthy for a solo project — zero open
issues and commits within the last week as of mid-2026. Versioning is
MAJOR.MINOR.COMMITS (not semver); the stated policy since 1.0 is accretive
and fixative changes only[^1].

## Getting Started

```clojure
;; deps.edn
{:deps {com.github.seancorfield/next.jdbc {:mvn/version "1.3.1118"}
        org.postgresql/postgresql        {:mvn/version "42.7.3"}}}
```

```clojure
(require '[next.jdbc :as jdbc])

(def ds (jdbc/get-datasource
          {:dbtype "postgresql" :dbname "app" :user "app" :password "..."}))

(jdbc/execute! ds ["create table address (id serial primary key, name text)"])

(jdbc/execute-one! ds ["insert into address (name) values (?)" "Ada"]
                   {:return-keys true})
;; => #:address{:id 1, :name "Ada"}

(jdbc/execute! ds ["select * from address where id = ?" 1])
;; => [#:address{:id 1, :name "Ada"}]  ; qualified keywords by default
```

## Architecture / How It Works

The library is a thin protocol layer over java.sql:

- **`Sourceable`** turns things (db-spec maps, JDBC URL strings) into a
  `DataSource`; **`Connectable`** yields `Connection`s. Because these are
  protocols, they extend to third-party pool objects and even via metadata.
  The deliberate path is db-spec → `DataSource` → `Connection`, steering
  users toward connection reuse[^1].
- **Three execution functions** cover everything: `plan` returns an
  `IReduceInit` that executes the SQL only when reduced and processes the
  live `ResultSet` with minimal allocation; `execute!` realizes a vector of
  hash maps; `execute-one!` realizes a single row (or
  `{:next.jdbc/update-count N}` for non-queries). This replaces the
  `query`/`execute!`/`db-do-commands` split that made `clojure.java.jdbc`
  confusing[^1].
- **Result-set builders** (`:builder-fn`) control row shape: qualified maps
  (default), unqualified maps, arrays, lower-case variants, or your own.
- **`datafy`/`nav` support is built in**: rows from `execute!` are
  navigable, so tools like Portal or Reveal can lazily walk foreign-key
  relationships out of a query result[^2].
- `prepare` exposes `PreparedStatement`s; `transact` /
  `with-transaction` manage transactions; `next.jdbc.sql` holds the
  syntactic-sugar `insert!`/`update!`/`delete!`/`find-by-keys` helpers,
  intentionally outside the core API.

There is no dialect abstraction. SQL strings pass through verbatim, and
database-specific behavior (streaming, generated keys, type coercion) is
handled by options and documented per-database quirks rather than by an
adapter layer.

## Production Notes

- **`get-datasource` does not pool.** The stock `DataSource` opens a fresh
  `Connection` per operation, which is fine at a REPL and slow in
  production. Use `next.jdbc.connection/->pool` with HikariCP or c3p0, and
  remember the pool object itself needs closing on shutdown[^2].
- **`plan` rows are not real maps.** Inside the reduction you get a
  lightweight view over the *current* `ResultSet` row; keyword access never
  builds a map. If a row escapes the reduction (returned, logged, assoc'd)
  you get errors or garbage. Call `next.jdbc.result-set/datafiable-row` (or
  `select-keys`) to realize a row you intend to keep[^2].
- **Nested transactions are a footgun.** JDBC has no true nested
  transactions. By default a `with-transaction` inside another one simply
  commits/rolls back on the same connection, which can silently commit the
  outer transaction's work early. `next.jdbc.transaction/*nested-tx*` can be
  set to `:ignore` or `:prohibit` to make the behavior explicit[^3].
- **Streaming large result sets is database-specific.** PostgreSQL requires
  `:fetch-size` plus a connection with auto-commit off; MySQL needs
  `useCursorFetch=true` and its own incantations. Without this, drivers
  buffer the entire result set in memory even under `plan`[^4].
- **Date/time coercion varies by driver.** Requiring `next.jdbc.date-time`
  enables reading SQL date/timestamp columns as Java Time types and adds
  parameter coercions some drivers lack. Skipping it is a common source of
  "works on H2, breaks on X" surprises[^4].
- **Upgrades are boring by design.** The accretion-only policy means jumps
  across the 1.3 line are expected to be drop-in; the real migration effort
  in this ecosystem is from `clojure.java.jdbc`, which has a dedicated
  guide[^5].

## When to Use / When Not

**Use when:**
- You are writing Clojure on the JVM against any JDBC-accessible database
  and want the community-standard, actively maintained option.
- You process large result sets and care about reduction without
  intermediate sequence allocation (`plan`).
- You want direct access to JDBC objects (`with-open`, prepared statements,
  driver options) instead of an abstraction that hides them.

**Avoid when:**
- You want an ORM-style entity layer, lifecycle hooks, or model definitions
  — this is a wrapper, not a data mapper.
- You expect SQL generation; `next.jdbc.sql` is minimal sugar, and complex
  queries mean hand-written SQL or a companion DSL.
- You are not on the JVM (ClojureScript has no JDBC) or your data store is
  not relational.

## Alternatives

- clojure/java.jdbc — the predecessor, in maintenance mode; use only when
  stuck on legacy code, and follow the official migration guide off it.
- seancorfield/honeysql — complement rather than replacement: generates SQL
  from Clojure data structures; most `next.jdbc` apps pair the two.
- metabase/toucan2 — use instead when you want a higher-level entity/model
  layer; it builds on `next.jdbc` and HoneySQL underneath.
- metosin/porsas — experimental performance-oriented JDBC mapping; use for
  benchmarking-grade hot paths, not as a maintained general library.
- funcool/clojure.jdbc — abandoned alternative wrapper from the mid-2010s;
  listed only so you avoid it in old tutorials.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-1.0 | 2019-01 | Repo created; alpha/beta cycle through spring 2019[^1]. |
| 1.0.0 | 2019-06-12 | "Gold" release; API declared stable after last rename (`reducible!` → `plan`)[^1]. |
| 1.2.674 | 2021 | First release under the `com.github.seancorfield` group ID after Clojars' verified-group policy; older releases live under `seancorfield`[^6]. |
| 1.3.x | 2022– | Current line; accretive changes only (batch execution, docs, driver quirk fixes)[^6]. |
| 1.3.1118 | 2026 | Latest release as of 2026-07; repo pushed 2026-07-17. |

## References

[^1]: next.jdbc README — motivation, release timeline, versioning policy.
      https://github.com/seancorfield/next-jdbc#motivation
[^2]: next.jdbc, "Getting Started" (cljdoc).
      https://cljdoc.org/d/com.github.seancorfield/next.jdbc/CURRENT/doc/getting-started
[^3]: next.jdbc, "Transactions".
      https://github.com/seancorfield/next-jdbc/blob/develop/doc/transactions.md
[^4]: next.jdbc, "Tips & Tricks" (per-database streaming and type quirks).
      https://github.com/seancorfield/next-jdbc/blob/develop/doc/tips-and-tricks.md
[^5]: next.jdbc, "Migrating from clojure.java.jdbc".
      https://cljdoc.org/d/com.github.seancorfield/next.jdbc/CURRENT/doc/migration-from-clojure-java-jdbc
[^6]: next.jdbc CHANGELOG.
      https://github.com/seancorfield/next-jdbc/blob/develop/CHANGELOG.md

## Tags

clojure, jvm, jdbc, sql, database, data-access, library, functional-programming, relational-database
