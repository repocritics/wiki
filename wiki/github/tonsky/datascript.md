# tonsky/datascript

> An immutable in-memory database and Datalog query engine for Clojure, ClojureScript, and JavaScript — "what if creating a database were as cheap as creating a hashmap?"

[GitHub repo](https://github.com/tonsky/datascript) ·
[API docs (cljdoc)](https://cljdoc.org/d/datascript/datascript) ·
[License: EPL-1.0](https://github.com/tonsky/datascript/blob/master/LICENSE)

## Overview

DataScript is Nikita Prokopov's (tonsky) 2014 reimplementation of Datomic's data model — datoms, Datalog queries, the Pull API, entities — as an ephemeral in-memory value[^1]. It shares no code with Datomic and drops its heavyweight parts: no durability by default, no history, no partitions, no full-text search, and a schema you only declare for attributes that need special behavior (cardinality-many, refs, uniqueness). The pitch is "database as a data structure": create a DB on page load, transact into it, query it with Datalog, throw it away when the tab closes.

The defining tradeoff is that DataScript optimizes for creation cost and API richness, not scale. Everything lives in persistent sorted sets in memory; queries are executed clause-by-clause with no query planner. For client-side app state — where the alternative is hand-filtering arrays — this is a strict upgrade. For datasets past a few hundred thousand datoms, load time, memory, and query latency all become visible, which is exactly the wall Roam Research and Logseq (the two highest-profile DataScript consumers[^1]) each hit at scale.

At ~5.8k stars it is one of the most-starred libraries in the Clojure ecosystem, and it spawned a family of forks that add what it deliberately omits (Datahike, Datalevin). It is a mature single-maintainer project: the API has been stable since 1.0 (2020), commits still land (last push October 2025), and the ~80 open issues reflect a maintainer who triages conservatively rather than a dead project.

## Getting Started

```clj
;; deps.edn
datascript/datascript {:mvn/version "1.7.8"}
```

```clj
(require '[datascript.core :as d])

(def schema {:aka {:db/cardinality :db.cardinality/many}})
(def conn (d/create-conn schema))

(d/transact! conn [{:db/id -1
                    :name  "Maksim"
                    :age   45
                    :aka   ["Max Otto von Stierlitz" "Jack Ryan"]}])

(d/q '[:find ?n ?a
       :where [?e :aka "Max Otto von Stierlitz"]
              [?e :name ?n]
              [?e :age  ?a]]
     @conn)
;; => #{["Maksim" 45]}
```

From JavaScript (no ClojureScript toolchain required): `npm install datascript`, then queries are passed as EDN strings and results come back as plain JS arrays[^1].

## Architecture / How It Works

Every fact is a **datom** — an `[entity attribute value tx]` tuple. A DB value is three sorted indexes over the same datoms: **EAVT** (entity-major, powers entity lookup and the Pull API), **AEVT** (attribute-major, powers "all values of attribute X"), and **AVET** (value-major, powers lookups by value and uniqueness checks)[^2]. The indexes are persistent sorted sets — a B+-tree-like immutable structure extracted into tonsky's separate `persistent-sorted-set` library[^3] — so a "new" DB after a transaction shares almost all structure with the old one. Old DB values remain valid forever; time travel and undo/redo are just holding references.

A `conn` is nothing more than an atom holding the latest DB value — `@conn` gives you the current immutable snapshot, `d/transact!` swaps in a new one and returns a tx-report (`:db-before`, `:db-after`, `:tx-data`) that you can itself query. Change tracking via `listen!` is a callback on that report, which is how Posh/re-posh build reactive UI bindings on top.

The query engine parses Datalog and executes clauses **in written order** as a series of hash joins against relations; there is no cost-based planner or clause reordering[^2]. Rules and recursion are supported; queries can run over multiple DBs and plain collections in the same `:where`. The Pull API and lazy `entity` objects provide the navigational alternative to Datalog.

Unlike Datomic, schema is not stored as datoms and is not queryable; attributes need no pre-declaration; values can be any type; and there is no `:db/txInstant` annotation or retained history — memory use is constant if you only update existing entities[^1]. Since 1.5, an opt-in storage protocol allows lazily loading index segments from durable backends (SQL implementations exist via `datascript-storage-sql`), turning it into a partially disk-backed store while keeping the in-memory programming model[^4].

## Production Notes

- **Clause order is your query planner.** Because there is no optimizer, putting an unselective clause first (e.g., `[?e :type ?t]` before a value filter) makes the engine materialize huge intermediate relations. Reordering clauses routinely turns 100ms queries into 1ms ones. This is the single most common DataScript performance mistake.
- **AVET is opt-in.** Direct index access by value (`d/datoms :avet ...`) throws unless the attribute is marked `:db/index true` or `:db/unique` — lookups by value on unindexed attributes fall back to scans.
- **shadow-cljs users must add externs.** Advanced compilation breaks DataScript's JS API without `:compiler-options {:externs ["datascript/externs.js"]}` — a years-old recurring trap (issues #432, #298)[^5].
- **Serialization is the real load-time cost.** Rehydrating a large DB on page load dominates startup; use `d/serializable`/`d/from-serializable` (fast JSON-safe path) or `datascript-transit` rather than pr-str/read-string, and consider shipping datoms and rebuilding indexes off the critical path.
- **Scale ceiling.** Comfortable at tens of thousands of datoms in a browser; at hundreds of thousands, GC pressure, initial load, and query latency degrade. Logseq's newer DB-based version pairs DataScript with durable storage rather than keeping everything resident — treat that as the pattern for big graphs.
- **No schema migration.** Changing schema semantics (e.g., cardinality) means creating a fresh DB with the new schema and re-transacting the datoms. Cheap by design, but it is on you.
- **Single-writer model.** `transact!` is an atom swap; there is no write concurrency story beyond what Clojure atoms give you. On the JVM under concurrent writers, contention serializes through the atom — fine for app state, wrong for a server-side OLTP store.
- **JS API is second-class ergonomically.** Queries as EDN strings mean no syntax checking until runtime, and keywords surface as strings (`":db/id"`). It works, but Clojure(Script) is the native habitat.

## When to Use / When Not

**Use when:**
- You need rich queries (joins, rules, recursion, aggregates) over client-side app state without a server round-trip.
- You want Datomic's programming model — DB as a value, tx-reports, Pull API — without Datomic's licensing, ops, or JVM-only constraint.
- You are building undo/redo, time-travel debugging, or optimistic sync, where immutable snapshots are the natural primitive.
- Your dataset fits comfortably in memory (roughly: thousands to low hundreds of thousands of datoms).

**Avoid when:**
- You need durability, history, or large datasets as first-class features — use Datahike, Datalevin, or XTDB instead of bolting storage on.
- Your queries are simple key lookups over flat data — a plain map or normalized atom is less machinery.
- You are outside the Clojure world and want idiomatic JS state management — the EDN-string query interface will fight you.
- You need server-grade concurrent writes or full-text search; both are explicitly out of scope[^1].

## Alternatives

- replikativ/datahike — a DataScript fork with durable storage and (optional) historical time-travel; use it when you want the same API but data that survives restarts.
- juji-io/datalevin — Datalog on LMDB, forked from DataScript then rewritten for durability and speed; use it for server-side or desktop apps where the DB must outlive the process and outgrow RAM.
- threatgrid/asami — an independent open-source triple store (CLJ/CLJS) with its own planner and optional durability; use it when you want analytics-flavored graph queries rather than Datomic compatibility.
- xtdb/xtdb — bitemporal, SQL+Datalog, document-oriented; use it when valid-time/transaction-time history is a requirement, not a trick.
- Datomic (proprietary, free binaries since 2023, not on GitHub as OSS) — use it when you actually want the full server-side system DataScript imitates: durability, history, distributed reads.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014-04 | Initial release; repo created 2014-04-15[^6]. |
| 0.x | 2015 | Pull API added (contributed by David Thomas Hume); feature set converges on Datomic parity[^1]. |
| 1.0 | 2020 | API declared stable; CLJ and CLJS on shared persistent-sorted-set indexes[^3]. |
| 1.5 | 2023 | Storage protocol: opt-in durable, lazily-loaded index backends[^4]. |
| 1.7.8 | 2025 | Current release line; maintenance-mode cadence, last push 2025-10. |

## References

[^1]: DataScript README — usage, feature list, "Differences from Datomic", projects using it. https://github.com/tonsky/datascript
[^2]: Nikita Prokopov, "DataScript internals" — datoms, EAVT/AEVT/AVET, query execution. https://tonsky.me/blog/datascript-internals/
[^3]: persistent-sorted-set — the B+-tree-like immutable index structure backing DataScript. https://github.com/tonsky/persistent-sorted-set
[^4]: DataScript storage documentation. https://github.com/tonsky/datascript/blob/master/docs/storage.md
[^5]: shadow-cljs externs requirement — issues #432, #298. https://github.com/tonsky/datascript/issues/432
[^6]: GitHub repository metadata (created 2014-04-15, EPL-1.0, ~5.8k stars as of 2026-07).

## Tags

clojure, clojurescript, javascript, database, datalog, in-memory, immutable, triple-store, client-side-state, query-engine
