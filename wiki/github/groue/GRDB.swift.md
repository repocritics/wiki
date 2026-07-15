# groue/GRDB.swift

> A SQLite toolkit for Swift — records, a type-safe query interface, and database-change observation, sitting close to the raw SQLite C library.

[GitHub repo](https://github.com/groue/GRDB.swift) ·
[License: MIT](https://github.com/groue/GRDB.swift/blob/master/LICENSE)

## Overview

GRDB is a Swift wrapper around SQLite that has been maintained by Gwendal Roué since 2015[^1]. It occupies a deliberate middle ground: higher-level than raw SQLite C calls or a thin FMDB-style wrapper, but lower-level and more explicit than an object graph like Core Data or Realm. You still think in terms of tables, rows, transactions, and SQL — GRDB just makes those things ergonomic and type-safe in Swift, and adds the pieces application code actually needs (records, migrations, change observation, concurrency).

The defining design choice is that GRDB never hides SQLite. Its `FetchableRecord` / `PersistableRecord` protocols generate persistence and fetch methods for your own structs, and its query interface builds SQL you could have written by hand — but raw SQL is a first-class citizen everywhere, and the library is explicit that you should learn SQLite rather than pretend it isn't there[^2]. This is the opposite tradeoff from Core Data (which abstracts the store away) and SwiftData (Apple's newer macro-driven layer): GRDB gives you control and predictability at the cost of writing schema and migrations yourself.

It is Apple-platform-centric. iOS/macOS/tvOS/watchOS are the supported, CI-tested targets; Linux is contributor-provided and explicitly not officially maintained or tested[^3]. That focus shows up everywhere — the concurrency model, SwiftUI integration via the companion GRDBQuery library, and app-group/app-extension database sharing guidance are all built for the Apple app-development reality.

## Getting Started

Swift Package Manager (the recommended path) — add `https://github.com/groue/GRDB.swift.git` as a dependency and link the `GRDB` product:

```swift
// Package.swift
.package(url: "https://github.com/groue/GRDB.swift.git", from: "7.0.0")
```

```swift
import GRDB

// 1. Open a connection
let dbQueue = try DatabaseQueue(path: "/path/to/db.sqlite")

// 2. Define schema
try dbQueue.write { db in
    try db.create(table: "player") { t in
        t.primaryKey("id", .text)
        t.column("name", .text).notNull()
        t.column("score", .integer).notNull()
    }
}

// 3. A record type — persistence is synthesized from Codable
struct Player: Codable, Identifiable, FetchableRecord, PersistableRecord {
    var id: String
    var name: String
    var score: Int
}

// 4. Write and read
try dbQueue.write { db in
    try Player(id: "1", name: "Arthur", score: 100).insert(db)
}
let best = try dbQueue.read { db in
    try Player.order(\.score.desc).limit(10).fetchAll(db)
}
```

Encryption uses a separate install path with SQLCipher; CocoaPods is frozen (see Production Notes).

## Architecture / How It Works

GRDB is a layer directly over the SQLite C API. The core abstractions:

- **Database connections** come in two shapes. `DatabaseQueue` serializes every access (reads and writes) through a single dispatch queue — simple, always correct, and the recommended default. `DatabasePool` opens the database in [WAL mode](https://www.sqlite.org/wal.html) and allows multiple concurrent reads alongside one writer, backed by a pool of reader connections. Both expose the same closure-based API (`read { db in ... }` / `write { db in ... }`); switching between them is mostly a one-line change.
- **The access closures are the safety boundary.** All database work happens inside `read`/`write` blocks that receive a `Database` value. That value must not escape the closure — GRDB 7 leans on Swift 6 strict concurrency and `Sendable` to catch escapes at compile time that earlier versions could only warn about at runtime[^4].
- **Records** are protocol-based, not a base class. `FetchableRecord` decodes rows into your type; `PersistableRecord` provides `insert`/`update`/`delete`/`upsert`. Conforming to `Codable` synthesizes both from your stored properties, so a plain struct becomes a persisted record with two protocol conformances and no boilerplate.
- **The query interface** is a Swift DSL (`Player.filter { $0.score > 100 }.order(\.name).fetchAll(db)`) that compiles to SQL. It covers most CRUD and joins via a typed associations system (`belongsTo`, `hasMany`, `hasOne`), but raw SQL and SQL interpolation are always available and freely mixable.
- **ValueObservation** is the change-tracking engine. It registers a `TransactionObserver` against SQLite's commit hooks, determines which tables/rows a tracked query touches, and re-runs the fetch when a committed transaction modifies that region — delivering fresh values via a callback, Swift concurrency `AsyncSequence`, a Combine publisher, or RxSwift (RxGRDB). Crucially the observation is *re-fetch based, not diff based*: on any relevant change it recomputes the whole observed value.
- **Migrations** are an ordered list of named registered closures plus a schema-version marker in the database. They run forward only — there is no automatic down-migration — and are the intended mechanism for evolving schema across app releases.

## Production Notes

**Concurrency is the thing to get right, and GRDB is opinionated about it.** The whole point of the closure API is to make it hard to touch the database off a serialized context. Reads inside `DatabasePool.read` see a consistent snapshot; writes serialize. The common mistakes are trying to hold a `Database` reference past the closure, or assuming a value fetched in one block is still current in another. GRDB 7's adoption of Swift 6 concurrency turns several of these into compile errors, which is why the 6→7 migration is non-trivial for apps with loose threading[^4].

**DatabasePool + WAL means sidecar files.** WAL mode creates `-wal` and `-shm` files next to your database. When sharing a database across processes (an app and its extensions in an App Group container), you must handle this deliberately — file coordination, `SQLITE_BUSY` retries, and the documented app-group setup — or you get corruption and busy errors. GRDB has a dedicated "Sharing a Database" guide precisely because this is a recurring source of production bugs[^5].

**ValueObservation can be expensive if you observe too much.** Because it re-fetches the entire tracked region on any change, observing a broad query over a hot table will recompute and redeliver frequently. Scope observations narrowly; don't observe a whole large table if you only render a page of it.

**Installation footguns.**
- **CocoaPods is stuck at 6.24.1.** A CocoaPods trunk issue prevents publishing new GRDB versions; to get GRDB 7 via CocoaPods you must point the Pod at a git branch or tag rather than a semantic version[^6]. Prefer SPM.
- **Carthage is unsupported** and has been explicitly declined by the maintainer[^7].
- **Encryption is a separate build.** Standard installs use the system SQLite; SQLCipher encryption requires the alternate install procedure and a custom SQLite build.
- **Linux is best-effort.** Not CI-tested, not officially maintained — fine for experimentation, risky as a production target[^3].

**Migrations are forward-only by design.** Plan schema changes as additive migrations; there is no built-in rollback. Test migrations against real old-version databases, since a broken migration ships to users who already have data.

## When to Use / When Not

**Use when:**
- You're building an Apple-platform app and want SQLite with Swift ergonomics, not an object graph you can't see through.
- You want database-change observation wired into SwiftUI (via GRDBQuery), Combine, or async/await.
- You value explicit schema, explicit migrations, and predictable SQL over automatic persistence magic.
- You already know SQLite (or want to) and occasionally need to drop to raw SQL.

**Avoid when:**
- You want the store fully abstracted away and are inside Apple's ecosystem anyway — Core Data or SwiftData fit that preference.
- You need first-class, officially supported Linux/server-side persistence — GRDB's Linux support is unofficial.
- You want automatic cross-device sync (CloudKit-style) out of the box — that's not GRDB's job.
- Your data model is a small key-value blob; a full SQL toolkit is overkill.

## Alternatives

- stephencelis/SQLite.swift — use instead when you want a lighter type-safe SQLite wrapper and don't need GRDB's observation, migrations, or record system.
- realm/realm-swift — use instead when you want an object database with built-in sync and are willing to adopt Realm's object model instead of SQL.
- ccgus/fmdb — use instead only for legacy Objective-C codebases already built on it; it's a thin, older SQLite wrapper with none of GRDB's Swift ergonomics.
- pointfreeco/sharing-grdb — use alongside/on top of GRDB when you want Point-Free's observation and dependency ergonomics; it builds on GRDB rather than replacing it.
- Apple Core Data / SwiftData — use instead when you prefer Apple's managed object graph and automatic persistence over explicit SQL and hand-written migrations.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2015 | First release; Swift SQLite toolkit by Gwendal Roué[^1]. |
| 5.0 | 2020 | Broad reference/API maturation era for records and observation. |
| 6.0 | 2022 | Major release; CocoaPods trunk frozen at 6.24.1 thereafter[^6]. |
| 7.0 | 2025 | Swift 6 strict-concurrency adoption; breaking migration from GRDB 6[^4]. |
| 7.11.1 | 2026-06-18 | Latest release as of this writing; requires Swift 6.1+, SQLite 3.20.0+[^8]. |

## References

[^1]: GRDB.swift README — "Proudly serving the community since 2015." https://github.com/groue/GRDB.swift
[^2]: GRDB.swift README, "What is GRDB?" — "Leverage your SQLite skills." https://github.com/groue/GRDB.swift#what-is-grdb
[^3]: GRDB.swift README, installation note — "Linux support is provided by contributors. It is not automatically tested, and not officially maintained." https://github.com/groue/GRDB.swift#swift-package-manager
[^4]: "Migrating From GRDB 6 to GRDB 7." https://github.com/groue/GRDB.swift/blob/master/Documentation/GRDB7MigrationGuide.md
[^5]: GRDB "Sharing a Database" guide — App Group containers, extensions, and file coordination. https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/databasesharing
[^6]: GRDB.swift README, CocoaPods note — last published version is 6.24.1 due to a CocoaPods issue. https://github.com/groue/GRDB.swift#cocoapods
[^7]: GRDB issue #433 — Carthage is unsupported. https://github.com/groue/GRDB.swift/issues/433
[^8]: GRDB.swift README, "Latest release" and requirements. https://github.com/groue/GRDB.swift/releases

## Tags

swift, sqlite, database, orm, ios, macos, persistence, sql, migrations, database-observation, apple-platforms
