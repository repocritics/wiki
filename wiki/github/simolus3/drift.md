# simolus3/drift

> A reactive, type-safe persistence library for Dart and Flutter that generates SQLite access code from Dart or SQL definitions.

[GitHub repo](https://github.com/simolus3/drift) ·
[Official website](https://drift.simonbinder.eu/) ·
[License: MIT](https://github.com/simolus3/drift/blob/develop/LICENSE)

## Overview

Drift is a persistence library that sits on top of SQLite and generates a
type-safe Dart API from your table and query definitions. You describe tables in
Dart classes or in `.drift` SQL files, run a build step, and get compile-time
checked query methods plus data classes. It is one of the most-used database
options in the Flutter ecosystem, alongside the lower-level `sqflite` and the
NoSQL-style `isar`/`hive`.

Two properties define drift. First, it is *reactive*: any `SELECT` can be turned
into a `Stream` that re-emits whenever a write touches one of the tables the
query reads. This maps cleanly onto Flutter's `StreamBuilder` and state
management patterns and is the main reason teams pick drift over raw `sqflite`.
Second, it is *code-generated*: nearly everything user-facing is produced by
`drift_dev` running under `build_runner`, which is also the source of drift's
biggest ergonomic tradeoff — you live with a compile/watch step, and generated
files (`*.g.dart`) are part of your tree.

The project was originally named **moor** ("room" reversed, after Android's Room
library). It was renamed to drift in October 2021 with the 1.0 release[^1]; the
old `moor` package is discontinued and its final version (4.6.1) is a
compatibility shim[^2]. Wording like "moor", `MoorDatabase`, or `.moor` files in
older tutorials refers to the same project under its previous name.

## Getting Started

Add the runtime, the native SQLite bindings, and the dev-time generator:

```yaml
# pubspec.yaml
dependencies:
  drift: ^2.0.0
  drift_flutter: ^0.2.0        # opens a database on device via sqlite3
dev_dependencies:
  drift_dev: ^2.0.0
  build_runner: ^2.0.0
```

```dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';

part 'database.g.dart';

class TodoItems extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get title => text().withLength(min: 1, max: 100)();
  BoolColumn get completed => boolean().withDefault(const Constant(false))();
}

@DriftDatabase(tables: [TodoItems])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(driftDatabase(name: 'app'));

  @override
  int get schemaVersion => 1;

  // Reactive: re-emits whenever TodoItems changes.
  Stream<List<TodoItem>> watchAll() => select(todoItems).watch();
}
```

Run the generator: `dart run build_runner build` (or `watch` during
development). This produces `database.g.dart` with the `TodoItem` data class and
the `_$AppDatabase` base.

## Architecture / How It Works

Drift is three packages developed in one repository, coordinated with melos[^3]:

- **`drift`** — the runtime. Query builder, streams, transactions, the
  connection/isolate machinery, and the platform database openers.
- **`drift_dev`** — the compiler. A `build_runner` builder that reads your table
  classes and `.drift` files, type-checks queries, and emits generated Dart. It
  also ships an analyzer plugin providing IDE lints on SQL.
- **`sqlparser`** — a pure-Dart SQL parser and static analyzer. It underpins
  drift's compile-time SQL checking and is published standalone for use without
  drift.

At runtime drift talks to an actual SQLite engine through the `sqlite3` package
(native FFI on Android/iOS/macOS/Windows/Linux) or a WebAssembly build of SQLite
in the browser. Drift itself does not embed a SQL engine; it builds statements,
executes them through a `QueryExecutor`, maps rows to generated classes, and
tracks which tables each statement touched.

The reactive layer is a table-level invalidation system, not a fine-grained
one. Drift records the set of tables a stream query reads; when a write reports
that it modified any of those tables, every stream subscribed to that table is
notified and **re-runs its full query**. There is no row-level diffing — a
single insert into a busy table re-executes every open stream that reads it.

Concurrency is drift's most distinctive feature. It can run the database on a
background isolate via `DriftIsolate`/`computeWithDatabase`, so query work does
not block the UI isolate. This is opt-in and adds setup complexity (the database
connection must be established over a port), but it is something most Dart
persistence libraries do not offer at all.

## Production Notes

- **Code generation is a standing cost.** Any table or query change requires
  re-running `build_runner`. On large schemas this is slow; `build_runner watch`
  is close to mandatory during development, and stale `.g.dart` files after a
  branch switch are a routine source of confusing errors. Commit generated files
  or generate in CI — teams disagree, but be consistent.

- **Stream queries scale by table, not by row.** Because any write to a table
  re-runs every stream reading it, apps with many always-on streams over a
  frequently-written table can spend real CPU re-executing queries. Narrow what
  each stream selects, avoid subscribing widgets to broad streams, and prefer
  one-shot `.get()` where reactivity is not needed.

- **Web/WASM setup is non-trivial.** Browser support requires serving a SQLite
  WASM binary and a drift worker, and choosing a storage backend (OPFS,
  IndexedDB). This is well-documented but materially more involved than the
  native path, and behavior differs across browsers[^4].

- **Migrations must be written and tested by hand.** Drift bumps `schemaVersion`
  and calls your `MigrationStrategy`, but the SQL steps are yours. `drift_dev`
  can export schema snapshots and generate migration test helpers; use them.
  Untested migrations that pass on a fresh install and fail on an upgraded one
  are the classic drift production incident.

- **SQLite version varies by platform.** On native, the bundled/system SQLite
  version depends on how you ship it (system library vs. bundled `sqlite3_flutter_libs`).
  Features like `RETURNING`, JSON1, or FTS5 availability differ; pin the bundle
  if you depend on a specific version or extension.

## When to Use / When Not

**Use when:**
- You want relational SQLite with compile-time-checked queries in a Flutter app.
- You want `SELECT` results as auto-updating streams wired into the UI.
- You need real SQL (joins, CTEs, window functions, FTS) rather than a key-value
  or document store.
- You want to move database work off the UI isolate.

**Avoid when:**
- You want zero code generation — `sqflite` (raw SQL, no codegen) or `isar`/`objectbox`
  (NoSQL, own generators) may fit better, though most typed options also generate.
- Your data is simple key-value settings — `shared_preferences` or `hive` is lighter.
- You need a non-SQLite backend (Postgres, a remote API) as the primary store;
  drift is SQLite-first, though `drift_postgres` exists for Postgres.
- You are outside Dart/Flutter — drift is Dart-only.

## Alternatives

- tekartik/sqflite — the low-level Flutter SQLite plugin. Use when you want raw
  SQL and no code generation and are willing to hand-map rows.
- isar/isar — NoSQL embedded database with its own query API. Use when you want a
  non-relational store and are comfortable with a smaller/rockier ecosystem.
- objectbox/objectbox-dart — object database focused on write/read throughput.
  Use when raw performance on object graphs matters more than SQL.
- hivedb/hive — pure-Dart key-value store. Use for simple local persistence
  without a SQL engine.
- vitusortner/floor — an annotation-based SQLite ORM inspired by Android Room.
  Use if you want a Room-like model and can accept a much smaller community.

## History

| Version | Date | Notes |
|---------|------|-------|
| moor 1.0.0 | 2019-03-09 | First release under the original name "moor"[^5]. |
| moor 4.6.1 | 2021-10-31 | Final moor release; now a compatibility shim to drift[^2]. |
| drift 1.0.0 | 2021-10-11 | Rename from moor to drift; new package name[^1]. |
| drift 2.0.0 | 2022-08-14 | Major version; API cleanups and generator changes[^6]. |
| drift 2.34.2 | 2026-07-14 | Recent 2.x release (latest at time of writing)[^6]. |

The project is actively maintained: releases land regularly and the repository
saw commits within a day of this page being written. It remains largely a
single-maintainer effort (Simon Binder, `simolus3`) with community contributions.

## References

[^1]: Drift 1.0 / moor rename announcement. https://drift.simonbinder.eu/name/
[^2]: `moor` package on pub.dev (discontinued, final 4.6.1). https://pub.dev/packages/moor
[^3]: Repository README, "Working on this project" — packages `drift`, `drift_dev`, `sqlparser`, managed with melos. https://github.com/simolus3/drift
[^4]: Drift documentation, "Web" / WASM setup. https://drift.simonbinder.eu/platforms/web/
[^5]: `moor` package version history on pub.dev (1.0.0 published 2019-03-09). https://pub.dev/packages/moor/versions
[^6]: `drift` package version history on pub.dev. https://pub.dev/packages/drift/versions

## Tags

dart, flutter, sqlite, persistence, orm, database, reactive, code-generation, type-safe, mobile
