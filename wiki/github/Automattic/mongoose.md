# Automattic/mongoose

> Schema-driven object modeling (ODM) for MongoDB in Node.js — adds structure, validation, and lifecycle hooks on top of a schemaless database.

[GitHub repo](https://github.com/Automattic/mongoose) ·
[Official website](https://mongoosejs.com) ·
[License: MIT](https://github.com/Automattic/mongoose/blob/master/LICENSE.md)

## Overview

Mongoose is the dominant ODM (Object-Document Mapper) for MongoDB on Node.js. First
released in 2010 by LearnBoost and later maintained under the Automattic org, it wraps
the official MongoDB Node.js driver with a schema layer, type casting, validation,
middleware ("hooks"), and query building[^1]. Its long-standing lead maintainer is
Valeri Karpov. At ~27k stars and 400+ contributors it is one of the most-depended-on
packages in the Node ecosystem, with tens of millions of npm downloads per week.

The defining tension is philosophical: MongoDB is schemaless by design, and Mongoose
puts a rigid schema back on top of it. This buys you validation, casting, populated
references (pseudo-joins), and pre/post hooks — but it also reintroduces much of the
ceremony NoSQL was meant to avoid, and adds a non-trivial abstraction layer between
your code and the wire protocol. Teams that want raw speed and full control over
BSON documents often drop to the native driver; teams that want an ORM-like developer
experience with a single source of truth for document shape reach for Mongoose.

Mongoose targets Node.js (and has alpha Deno support[^2]). Version 9.0.0 shipped on
2025-11-21 with breaking changes[^3].

## Getting Started

```sh
npm install mongoose
```

```js
const mongoose = require('mongoose');

const blogSchema = new mongoose.Schema({
  title:  { type: String, required: true },
  body:   String,
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  tags:   [String],
  date:   { type: Date, default: Date.now },
});

blogSchema.pre('save', function (next) {
  this.title = this.title.trim();
  next();
});

const Blog = mongoose.model('Blog', blogSchema);

async function main() {
  await mongoose.connect('mongodb://127.0.0.1:27017/my_database');
  const post = await Blog.create({ title: 'Hello', body: 'World' });
  const found = await Blog.findById(post._id).populate('author');
  console.log(found.title);
}
main();
```

Note the collection-naming rule: `mongoose.model('Ticket', schema)` writes to the
**`tickets`** collection — Mongoose lowercases and pluralizes the model name unless you
override it.

## Architecture / How It Works

Mongoose sits on top of `mongodb` (the official Node driver)[^4]. The core objects:

- **Schema** — declares paths, types, defaults, validators, indexes, virtuals, methods,
  statics, and hooks. A schema is pure metadata; it holds no connection.
- **Model** — a compiled constructor bound to a connection and a collection. Static
  query methods (`find`, `findOne`, `updateOne`, `aggregate`) live here.
- **Document** — a hydrated instance with change tracking. Mongoose records which paths
  were modified (`isModified`) so `save()` emits a minimal `$set`/`$unset` update, not a
  full replace.
- **Query** — a thenable builder. `Blog.find({...})` returns a `Query`, not a Promise;
  it executes when awaited or when `.exec()` is called.
- **Connection** — wraps a driver connection pool. `mongoose.connect` populates the
  default global connection; `mongoose.createConnection` returns an isolated one.

**Command buffering** is a signature behavior: Mongoose queues model operations issued
before the connection is established and flushes them once connected. Convenient in
scripts, but it hides connection failures — a wrong URI can leave queries hanging until
a buffer timeout rather than erroring immediately.

**Middleware** runs pre/post on document ops (`save`, `validate`, `remove`), query ops
(`find`, `updateOne`), and aggregate. Query middleware `this` is the Query, not the
document — a frequent source of bugs when authors expect `this` to be the record being
updated.

**`populate`** performs application-side joins: a second query resolves `ObjectId`
references into embedded objects. It is not a server-side `$lookup`, so N referenced
collections can mean N extra round trips unless batched.

## Production Notes

- **`lean()` for read paths.** Full hydration into Mongoose Documents is expensive
  (getters, virtuals, change tracking). `Model.find().lean()` returns plain objects and
  is dramatically faster and lighter for read-only endpoints. This is the single most
  impactful Mongoose performance lever.
- **The `strictQuery` footgun.** Historically, filter fields not in the schema were
  silently dropped from queries, which could turn `find({ notAField: x })` into
  `find({})` — returning the whole collection. The default flipped across v6/v7; pin
  `mongoose.set('strictQuery', ...)` explicitly rather than relying on the version
  default.
- **Connection pool sizing.** The default `maxPoolSize` (driver default 100) is often
  wrong for serverless. In Lambda/Vercel functions, reuse a cached connection across
  invocations and lower the pool size; do not `connect` per request.
- **Buffering masks outages.** With `bufferCommands` on (default), a dropped MongoDB
  connection buffers writes silently until `bufferTimeoutMS` (default 10s) expires.
  Consider disabling buffering in services that must fail fast.
- **TypeScript friction.** Mongoose's generated types are notoriously involved; inferred
  document types, `HydratedDocument`, and method/static typing require care. Many TS
  teams layer `typegoose` or hand-write interfaces.
- **Transactions** require a replica set (even single-node) — they do not work against a
  standalone `mongod`. Sessions must be threaded through every operation manually.
- **Upgrade pain.** v7 (2023) removed callback support entirely — every `cb`-style call
  had to become a Promise[^5]. v6 (2021) moved to driver 4.x and changed several
  connection defaults. Read each major's migration guide; they are not cosmetic.

## When to Use / When Not

**Use when:**
- You want enforced document structure, validation, and casting on top of MongoDB.
- You lean on lifecycle hooks (audit fields, cascade cleanup, derived data).
- Your team wants an ORM-like experience and a single schema definition per collection.
- You use references and want ergonomic `populate` instead of hand-written `$lookup`.

**Avoid when:**
- You need maximum throughput and minimal overhead — the native driver is faster.
- Your documents are genuinely polymorphic/schemaless and a rigid schema fights you.
- You want compile-time-first type safety as the primary contract — Prisma or MikroORM
  model that more directly.
- You are on a non-Mongo store, or want one abstraction across SQL and Mongo.

## Alternatives

- mongodb/node-mongodb-native — the official driver Mongoose wraps; use it directly when you want zero schema overhead and full control of BSON.
- prisma/prisma — schema-first, codegen'd type-safe client with a MongoDB connector; use when compile-time types and migrations matter more than hooks/populate.
- typegoose/typegoose — TypeScript class + decorator layer *on top of* Mongoose; use when you want Mongoose semantics with class-based, typed schemas.
- mikro-orm/mikro-orm — TS-first ORM with a unit-of-work/identity-map and MongoDB support; use when you want entity tracking closer to a classic ORM.
- mongodb/mongoose-alternatives via the driver + Zod — use hand-rolled validation when you want schema checks without the full ODM.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010 | Initial release by LearnBoost. |
| 4.0 | 2015-03 | Query middleware, discriminators, major API cleanup. |
| 5.0 | 2018-01 | MongoDB driver 3.x, native promises, change streams. |
| 6.0 | 2021-08 | Driver 4.x, removed legacy connect options, `strictQuery` default change. |
| 7.0 | 2023-02 | Callback support removed — Promises only[^5]. |
| 8.0 | 2023-10 | Driver 6.x, Node 16+ baseline, search index helpers. |
| 9.0.0 | 2025-11-21 | Latest major; breaking changes per migration guide[^3]. |

## References

[^1]: Mongoose README and documentation. https://mongoosejs.com/docs/
[^2]: Mongoose README — "supports Node.js and Deno (alpha)." https://github.com/Automattic/mongoose
[^3]: Mongoose docs, "Migrating to Mongoose 9" — 9.0.0 released 2025-11-21. https://mongoosejs.com/docs/migrating_to_9.html
[^4]: MongoDB Node.js driver. https://github.com/mongodb/node-mongodb-native
[^5]: Mongoose docs, "Migrating to Mongoose 7" — callback removal. https://mongoosejs.com/docs/migrating_to_7.html

## Tags

javascript, typescript, nodejs, mongodb, odm, orm, database, schema, data-modeling, deno
