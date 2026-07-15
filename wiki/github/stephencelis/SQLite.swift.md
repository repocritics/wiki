# stephencelis/SQLite.swift

> A type-safe Swift DSL over SQLite3 — compile-time-checked SQL as an expression builder, not an ORM.

[GitHub repo](https://github.com/stephencelis/SQLite.swift) ·
[License: MIT](https://github.com/stephencelis/SQLite.swift/blob/master/LICENSE.txt)

## Overview

SQLite.swift is a thin, type-safe Swift layer over the SQLite3 C library. It has two faces: a fluent, generic-typed query DSL (`Table`, `Expression<T>`, `filter`, `insert`, the `<-` setter operator) that builds SQL at compile time with optional-awareness, and a lightweight wrapper over the raw C API (`db.prepare("SELECT ...")`, positional binding) for when the DSL is not expressive enough[^1]. It is not an ORM: there is no relationship graph, no change tracking, no object observation, no managed object context. You model tables and columns explicitly and read rows into typed values.

The repository was created in October 2014, only months after Swift's public debut, which makes it one of the oldest surviving Swift database libraries[^2]. That longevity is its main asset and the source of its defining tension. The library has remained on a pre-1.0 `0.x` version line for over a decade, and its release cadence is sporadic — there was a gap of more than two years between 0.12.2 (June 2019) and 0.13.0 (August 2021)[^3]. In practice the API is mature and stable, but the semver signal and the slow maintenance pace mean you are adopting a well-worn tool, not an actively-racing one.

The audience is Apple-platform developers (iOS, macOS, tvOS, watchOS; Linux with limitations) who want SQL they can see and control, with the compiler catching column-type and nullability mistakes, and who do not want the weight or magic of Core Data / Realm.

## Getting Started

Add via Swift Package Manager in `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/stephencelis/SQLite.swift.git", from: "0.16.0")
]
```

A minimal typed usage:

```swift
import SQLite

let db = try Connection("path/to/db.sqlite3")

let users = Table("users")
let id    = SQLite.Expression<Int64>("id")
let name  = SQLite.Expression<String?>("name")   // Optional column
let email = SQLite.Expression<String>("email")

try db.run(users.create { t in
    t.column(id, primaryKey: true)
    t.column(name)
    t.column(email, unique: true)
})

let rowid = try db.run(users.insert(name <- "Alice", email <- "alice@mac.com"))

for user in try db.prepare(users.filter(id == rowid)) {
    print(user[id], user[name] ?? "-", user[email])
}
```

Note the `SQLite.Expression` qualification: bare `Expression` collides with `SwiftUI.Expression`, so codebases importing SwiftUI must namespace it explicitly[^1]. Also available via CocoaPods (`pod 'SQLite.swift'`) and Carthage.

## Architecture / How It Works

The DSL is built almost entirely on Swift operator overloading and generics. `Expression<T>` is a phantom-typed wrapper carrying a SQL fragment plus its bindings; overloaded operators (`==`, `<`, `&&`, `+`, `like`) compose those fragments into larger typed expressions. `Table.filter(...)`, `.select(...)`, `.order(...)`, `.limit(...)` return new query values (chainable, lazy — no SQL runs until you `prepare`/`run`/`scalar`). At execution the query renders to a parameterized SQL string with a bindings array, which is handed to the C layer.

Underneath, `Connection` wraps a `sqlite3*` handle. It serializes all access through an internal queue, so a single `Connection` is safe to share across threads, but it offers no async/await interface and no reactive observation — you drive it synchronously. Statements are prepared via `sqlite3_prepare_v2` and values bound positionally; row values are pulled out and cast to the Swift type the caller declared. The optional-awareness (`Expression<String?>` vs `Expression<String>`) is what lets the type system distinguish nullable columns and force you to unwrap.

Additional capabilities layered on top: schema query and migration helpers, FTS4/FTS5 full-text search, custom SQL functions and collations bridged to Swift closures, and SQLCipher support (encrypted databases) which is wired in through the Swift Package Manager build only[^1]. `Codable` row mapping exists but is a convenience over the same primitives, not a full object mapper.

## Production Notes

- **Type-checker blowups.** The operator-heavy DSL is the classic Swift trap: long chained expressions can push the compiler into exponential type inference, producing "unable to type-check this expression in reasonable time" errors or minute-long build stalls. The fix is to break expressions into intermediate `let` bindings with explicit types. This is the single most-reported friction and is inherent to the design, not a bug.
- **Pre-1.0 for a decade.** Despite production use at scale, the project has never cut a 1.0. Treat the `0.x` line as de-facto stable but pin exact versions; minor bumps have occasionally changed API surface.
- **Sporadic maintenance.** Releases cluster then go quiet for long stretches (the 2019→2021 gap being the starkest)[^3]. Bugfixes and new-Swift-version support can lag. As of mid-2026 the repo is still receiving commits, so it is maintained, just not fast-moving.
- **Not an ORM — by design.** No relationships, cascade rules, lazy-loaded associations, or change observation. If you find yourself hand-building a join/identity layer on top, you have outgrown the library and want GRDB or Core Data.
- **Threading.** The internal serial queue makes a shared `Connection` safe, but there is no connection pool and no built-in read/write concurrency (SQLite WAL mode helps, and the library exposes `journalMode`/WAL configuration, but you manage concurrency strategy yourself).
- **Linux.** Supported with documented limitations — some Foundation-dependent and platform-specific pieces behave differently or are unavailable[^1]. Verify on-target rather than assuming macOS parity.
- **SQLCipher.** Encryption is only available through the SPM integration path, not the plain framework — a common surprise for teams on Carthage/CocoaPods.

## When to Use / When Not

**Use when:**
- You want to write SQL you can read and reason about, with compile-time column/type checking and nullability enforcement.
- You want a light dependency over stock SQLite without Core Data's object graph or Realm's separate storage engine.
- Your data model is table-and-query shaped, not object-graph shaped.

**Avoid when:**
- You need relationships, change observation, or reactive queries out of the box — GRDB or Core Data fit better.
- Your build is already type-check-bound and you cannot afford the DSL's inference cost.
- You want an actively fast-moving library with frequent releases and a stable 1.0 contract.
- You need first-class cross-platform (server-side Linux) parity as a hard requirement.

## Alternatives

- groue/GRDB.swift — the most common modern alternative; richer, more actively maintained, adds record protocols, query observation (`ValueObservation`), and async APIs. Use when you want observation, relationships, or a faster release pace.
- ccgus/fmdb — the older Objective-C wrapper over SQLite. Use when you are in a mixed ObjC/Swift codebase or want a battle-tested, minimal, non-DSL layer.
- realm/realm-swift — an object database, not a SQLite wrapper. Use when you want a fully managed object store with live objects and built-in sync, and do not need raw SQL.
- pointfreeco/swift-structured-queries — a newer type-safe query DSL (pairs with GRDB). Use when you want macro-driven, compile-checked SQL with modern Swift tooling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9.0 | 2014 (created 10-04) | Early releases; among the first Swift SQLite libraries[^2]. |
| 0.11.6 | 2019-04-19 | Swift 4/5-era line. |
| 0.12.0 | 2019-04-24 | Feature line before a long maintenance gap[^3]. |
| 0.13.0 | 2021-08-22 | First release after a 2+ year gap[^3]. |
| 0.14.1 | 2022-11-02 | Continued 0.13/0.14 stabilization. |
| 0.15.0 | 2024-02-24 | Swift 5.x, SPM-centric packaging, SQLCipher via SPM. |
| 0.15.5 | 2026-01-22 | 0.15 maintenance line. |
| 0.16.0 | 2026-03-08 | Latest release as of mid-2026[^3]. |

## References

[^1]: SQLite.swift README and Documentation — features, usage, `SQLite.Expression` naming note, SQLCipher-via-SPM, and Linux limitations. https://github.com/stephencelis/SQLite.swift
[^2]: Repository metadata (created 2014-10-04), GitHub API. https://api.github.com/repos/stephencelis/SQLite.swift
[^3]: Release history and tags (dates for 0.12.2→0.13.0 gap and 0.16.0), GitHub API. https://github.com/stephencelis/SQLite.swift/releases

## Tags

swift, sqlite, database, query-builder, type-safe, ios, macos, dsl, sqlcipher, persistence, orm-alternative, apple-platforms
