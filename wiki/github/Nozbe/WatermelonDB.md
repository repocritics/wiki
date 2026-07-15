# Nozbe/WatermelonDB

> A lazy, observable SQLite layer for React and React Native — query on a native thread, render only what you touch.

[GitHub repo](https://github.com/Nozbe/WatermelonDB) ·
[Official website](https://watermelondb.dev) ·
[License: MIT](https://github.com/Nozbe/WatermelonDB/blob/master/LICENSE)

## Overview

WatermelonDB is a reactive persistence layer built on top of SQLite, aimed squarely at React Native apps that hold thousands to tens of thousands of records. It was created at Nozbe by Radek Pietruszewski and has backed the Nozbe apps in production since 2017[^1]; the repository was opened in August 2018[^2]. The pitch is narrow and specific: keep app launch fast regardless of database size by loading nothing eagerly, running all queries on the native SQLite database on a separate thread, and making the results observable so the UI re-renders automatically when the underlying rows change.

The defining tradeoff is laziness versus convenience. Loading an entire dataset into a JavaScript store (Redux, MobX, AsyncStorage-backed state) is simple but scales badly — cold start slows as the store grows, especially on low-end Android. Watermelon avoids that by keeping data in SQLite and only materializing records on demand, but you pay for it with a decorator-heavy model layer, an append-only migration discipline, and — the big one — a sync engine that ships only the client half. You bring your own backend.

Notably, after seven-plus years the library is still pre-1.0 (0.x). It is stable and widely deployed (Mattermost, Rocket.Chat, and others ship it), but the version number honestly reflects an API that still shifts between minor releases[^3].

## Getting Started

```bash
npm install @nozbe/watermelondb
# React Native also needs the native module linked (autolinking on RN 0.60+)
# iOS: cd ios && pod install
```

```js
// schema.js
import { appSchema, tableSchema } from '@nozbe/watermelondb'

export default appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'posts',
      columns: [
        { name: 'title', type: 'string' },
        { name: 'body', type: 'string' },
        { name: 'created_at', type: 'number' },
      ],
    }),
  ],
})
```

```js
// Post.js — a Model maps to one table
import { Model } from '@nozbe/watermelondb'
import { field, date, children } from '@nozbe/watermelondb/decorators'

export default class Post extends Model {
  static table = 'posts'
  static associations = { comments: { type: 'has_many', foreignKey: 'post_id' } }

  @field('title') title
  @date('created_at') createdAt
  @children('comments') comments
}
```

```js
// All writes go through a writer block; records are otherwise immutable.
await database.write(async () => {
  await database.get('posts').create(post => {
    post.title = 'Hello'
  })
})
```

## Architecture / How It Works

The stack is layered: `Database` → `Collection` (one per table) → `Model` (one per row) → decorated `@field` accessors that read from an in-memory `_raw` object synced to SQLite. Queries are built with a chainable `Q` DSL (`Q.where`, `Q.on`, `Q.sortBy`) that compiles to SQL rather than filtering in JS.

The performance story rests on the **adapter** boundary. On native, `SQLiteAdapter` runs queries against real SQLite off the JS thread. With the JSI option enabled it calls into the native database synchronously through the JavaScript Interface, skipping the React Native async bridge — this is where most of the speed comes from and is the recommended configuration. On the web there is no SQLite; `LokiJSAdapter` runs an in-memory JS database persisted to IndexedDB. That web adapter loads the working set into memory, which quietly undoes the lazy-loading premise — web is a real but second-class target.

Reactivity is built on RxJS observables. `Query.observe()` and `Model.observe()` emit whenever matching rows change; the React bindings (`withObservables` HOC, now shipped under `@nozbe/watermelondb/react`) subscribe a component to those observables and re-render on emission. Because observation is row/query-scoped, a change anywhere in the app fans out only to the components that actually depend on it.

Schema changes are handled through **append-only migrations**. You bump `schema.version`, add a migration step describing the delta, and never edit past migrations — the same discipline as a server-side migration tool, applied to an on-device database.

## Production Notes

**Sync is the hard part, and it is your job.** WatermelonDB ships `synchronize()` with a `pullChanges`/`pushChanges` protocol, but no server. You implement the backend endpoints, the change-tracking, and the conflict policy. The built-in resolution is last-write-wins merged at the column level via per-record `_changed` tracking — adequate for many apps, but anything needing real conflict resolution (collaborative editing, inventory) requires design work Watermelon does not do for you[^4]. Getting the server's `changes` payload shape and `last_pulled_at` handling exactly right is the most common source of production bugs.

**Everything mutating must run inside `database.write()`.** Reads are free; creates, updates, and deletes outside a writer throw. Batching many operations with `database.batch(...)` or `prepareCreate`/`prepareUpdate` is not optional at scale — issuing thousands of individual writes serially is a known slowdown.

**Decorators require build config.** Models depend on legacy/experimental decorators, so Babel (`@babel/plugin-proposal-decorators` with `legacy`) or `experimentalDecorators` in TypeScript must be set. The codebase was written Flow-first; TypeScript definitions exist and are maintained but occasionally lag the Flow source of truth.

**Observation leaks.** Subscriptions you create manually (outside `withObservables`) must be disposed. Long-lived observers on large queries retain results in memory; forgetting to unsubscribe is a slow leak that surfaces as growing memory on long sessions.

**Deletes and sync interact.** Records are marked deleted (`markAsDeleted`) rather than destroyed until sync confirms, so "deleted" rows linger locally. `unsafeResetDatabase` exists but, as named, is a footgun.

## When to Use / When Not

**Use when:**
- You have a React Native app whose local dataset is large enough that a JS-store approach slows launch.
- You want automatic, fine-grained UI reactivity tied to persisted data.
- You are willing to build and own the sync backend, and want offline-first behavior on your terms.

**Avoid when:**
- Your app holds a small amount of data — Redux/MobX with a persistence adapter is far less machinery.
- You are web-first; the LokiJS adapter forfeits the core advantage and other libraries serve the browser better.
- You want turnkey sync (a hosted service or a managed backend) rather than a protocol you implement yourself.
- You want raw SQL access without an ORM, decorators, and an observation layer in the way.

## Alternatives

- realm/realm-js — object database for React Native with historically bundled sync; choose it when you want objects instead of SQL, but note MongoDB's deprecation of Atlas Device Sync.
- powersync-ja/powersync-js — SQLite-backed offline-first with a hosted sync service; choose it when you do not want to build and operate the sync backend yourself.
- pubkey/rxdb — reactive, offline-first, web-first database with pluggable replication; choose it for PWA/browser-primary apps where Watermelon's web story is weak.
- OP-Engineering/op-sqlite — fastest raw SQLite for React Native via JSI; choose it when you want direct SQL and no ORM or reactivity layer.
- drizzle-team/drizzle-orm — type-safe SQL query builder (pairs with expo-sqlite); choose it when you want typed SQL without an observation/decorator model.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017 | Powers Nozbe in production before public release[^1]. |
| 0.x (public) | 2018-09 | Repository opened and open-sourced[^2]. |
| 0.x | ~2021 | `database.action()` split into `database.write()` / `database.read()`. |
| 0.x | ongoing | JSI adapter path for synchronous native SQLite; sync engine refinements; React bindings folded into `@nozbe/watermelondb/react`. |
| 0.x | 2025-08 | Still pre-1.0; last push 2025-08-11, ~11.7k stars[^3]. |

## References

[^1]: WatermelonDB README — "Proven. Powers Nozbe since 2017." https://github.com/Nozbe/WatermelonDB#readme
[^2]: GitHub API, `repos/Nozbe/WatermelonDB` — `created_at` 2018-08-28. https://github.com/Nozbe/WatermelonDB
[^3]: GitHub API, `repos/Nozbe/WatermelonDB` — 11,747 stars, 651 forks, last push 2025-08-11, still 0.x versioning (fetched 2026-07). https://github.com/Nozbe/WatermelonDB
[^4]: WatermelonDB docs — Synchronization. https://watermelondb.dev/docs/Sync/Intro

## Tags

javascript, react-native, react, database, sqlite, offline-first, reactive, orm, rxjs, sync, persistence, mobile
