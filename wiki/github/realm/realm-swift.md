# realm/realm-swift

> An embedded, object-oriented mobile database for Apple platforms — a Core Data / SQLite replacement with live, auto-updating objects.

[GitHub repo](https://github.com/realm/realm-swift) ·
[Official website](https://realm.io) ·
[License: Apache-2.0](https://github.com/realm/realm-swift/blob/community/LICENSE)

## Overview

Realm is an embedded database that runs in-process on iOS, macOS, tvOS and watchOS. This repository holds the Swift and Objective-C SDKs; the actual storage engine lives in the separate C++ project realm/realm-core, which the SDKs wrap. Rather than mapping objects to rows through an ORM, Realm persists object graphs directly and hands back thin accessors backed by a memory-mapped file, so a fetched object is a live view of committed state rather than a detached copy[^1].

The project began in 2012 and launched publicly in 2014 as the flagship product of Realm Inc., a Y Combinator company. MongoDB acquired Realm in 2019 and folded the local database into a cloud-sync product line (Realm Cloud → MongoDB Realm → Atlas Device Sync)[^2]. That coupling defines the repo's current tension: the local database is mature and widely deployed, but its headline differentiator — transparent bidirectional sync to a managed backend — has been deprecated by its own owner (see Production Notes). The default branch is named `community`, reflecting a shift toward community-maintained status after MongoDB wound down active feature work.

The defining tradeoff is the live-object model. It eliminates a large amount of boilerplate and keeps UI reactive for free, but it makes Realm objects thread-confined and tied to a database version, which is the single most common source of crashes and confusion for new users.

## Getting Started

```
// Swift Package Manager — add to Package.swift dependencies:
.package(url: "https://github.com/realm/realm-swift.git", from: "20.0.0")
// then depend on the "RealmSwift" product.
// Also available via CocoaPods (pod 'RealmSwift') and Carthage.
```

```swift
import RealmSwift

class Dog: Object {
    @Persisted var name: String
    @Persisted var age: Int
}

let realm = try! Realm()           // opens/creates the default .realm file

try! realm.write {                 // all mutations happen in a write transaction
    realm.add(Dog(value: ["name": "Rex", "age": 1]))
}

// Results are lazy, live, and auto-updating — no manual refetch.
let puppies = realm.objects(Dog.self).where { $0.age < 2 }
print(puppies.count)
```

## Architecture / How It Works

There are three layers. **Realm Core** (C++, realm/realm-core) is the storage engine: a memory-mapped file with a custom B-tree layout and multiversion concurrency control (MVCC). Readers see a stable snapshot of a committed version while a single writer commits the next; this is what makes reads zero-copy and lock-free. **Realm Objective-C** (`RLMObject`, `RLMRealm`) binds Core to the Objective-C runtime. **RealmSwift** is the idiomatic Swift surface layered on top of the Objective-C classes.

Persisted properties are not stored in the Swift object at all. The modern `@Persisted` property wrapper (introduced in Realm 10.10, replacing the older `@objc dynamic var` convention) intercepts every get/set and reads or writes straight through to the underlying row in the mapped file[^3]. This is why objects are "live": two accessors to the same primary key observe each other's writes immediately, with no notification plumbing.

Collections (`Results`, `List`, `LinkingObjects`) are equally lazy — they are queries, not materialized arrays, and they re-run against the current version on access. Change notifications deliver fine-grained index sets (insertions / deletions / modifications) suitable for driving `UITableView`/`UICollectionView` diffing, and SwiftUI integration (`@ObservedResults`, `@StateRealmObject`) is built on top of that mechanism plus **frozen objects** — immutable, thread-safe snapshots detached from the live version.

Because a `Realm` instance is pinned to a thread and a version, objects obtained from it may not be passed to another thread. Crossing threads requires `ThreadSafeReference`, a frozen copy, or re-opening the Realm and re-querying by primary key on the destination thread.

## Production Notes

**Atlas Device Sync is sunset.** MongoDB announced deprecation of Atlas Device Sync in September 2024 with end-of-life in September 2025[^4]. The local database in this repo continues to function, but teams that adopted Realm specifically for turnkey device-to-cloud sync must migrate to another backend or self-managed replication. Treat "Realm the local store" and "Realm sync" as separate bets when evaluating today.

**Thread confinement is the top footgun.** Passing a live object, `Results`, or a `Realm` across a thread boundary throws or crashes. Background work must open its own Realm on its own thread (or actor) and hand results back as frozen objects or primary keys. This trips up nearly every new codebase.

**File growth and MVCC pinning.** MVCC retains old versions until every reader referencing them is gone. A long-lived Realm on a background thread — or a notification token never invalidated — can pin an old version and cause the file to grow unbounded. Mitigations: invalidate stale Realms, keep write transactions short, and use `shouldCompactOnLaunch` to reclaim space.

**Migrations are mandatory and manual.** Any change to a persisted schema (adding/removing/renaming a property or model) requires bumping `schemaVersion` and, for non-additive changes, supplying a migration block. Ship an untested migration and you get a hard open failure in the field.

**Other operational notes:** encryption uses a caller-supplied 64-byte AES-256 key that you must store in the Keychain yourself; notifications need an active run loop; write transactions are globally serialized, so a slow write blocks all writers; and the Objective-C/C++ core adds meaningful binary size versus a thin SQLite wrapper. Building from source requires Xcode 15.3 or newer and downloads a prebuilt core binary on first build.

## When to Use / When Not

**Use when:**
- You want reactive, auto-updating models wired to SwiftUI/UIKit with minimal glue code.
- Your data is an object graph with relationships rather than tabular/relational data you want to query in SQL.
- You need on-device encryption and offline-first persistence out of the box.
- You are willing to design around thread confinement.

**Avoid when:**
- You adopted Realm primarily for Atlas Device Sync — that backend is being retired.
- You want SQL, arbitrary ad-hoc queries, or full control over the on-disk format.
- You need to freely pass model objects across threads/actors without ceremony.
- You prefer to stay entirely on Apple's first-party stack (Core Data / SwiftData).

## Alternatives

- groue/GRDB.swift — SQLite toolkit for Swift; use when you want real SQL, explicit migrations, and no live-object/thread-confinement model.
- stephencelis/SQLite.swift — lightweight, type-safe SQLite wrapper; use when you want a thin query builder over plain SQLite.
- couchbase/couchbase-lite-ios — embedded NoSQL database with a still-supported sync backend; use when transparent device-to-cloud sync is the requirement now that Atlas Device Sync is ending.
- realm/realm-core — the underlying C++ engine; relevant if you need Realm semantics outside Swift (e.g. the Kotlin or JS SDKs) or are debugging storage-level behavior.
- Apple SwiftData / Core Data (not in this wiki, first-party, closed source) — use when staying on Apple's own persistence stack matters more than cross-platform parity.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012-04 | Repository created; early Objective-C-era development[^1]. |
| public launch | 2014-07 | Realm launches publicly as a Core Data/SQLite alternative. |
| — | 2019 | MongoDB acquires Realm Inc.[^2] |
| 10.0 | 2020-10 | MongoDB Realm sync, frozen objects, embedded objects; new file format. |
| 10.10 | 2021 | `@Persisted` property wrapper replaces `@objc dynamic var`[^3]. |
| — | 2024-09 | MongoDB announces Atlas Device Sync deprecation (EOL 2025-09)[^4]. |

## References

[^1]: Realm Swift README and repository metadata (repo created 2012-04-16). https://github.com/realm/realm-swift
[^2]: MongoDB, "MongoDB Acquires Realm" (2019). https://www.mongodb.com/company/newsroom/press-releases/mongodb-acquires-realm
[^3]: Realm Swift release notes, `@Persisted` introduced in 10.10.0. https://github.com/realm/realm-swift/blob/community/CHANGELOG.md
[^4]: MongoDB, "Atlas Device Sync Deprecation." https://www.mongodb.com/docs/atlas/app-services/sync/device-sync-deprecation/

## Tags

swift, objective-c, ios, mobile-database, embedded-database, offline-first, orm-alternative, reactive, apple-platforms, mongodb, sync
