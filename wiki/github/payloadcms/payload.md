# payloadcms/payload

> A TypeScript, config-driven headless CMS and application framework that runs natively inside a Next.js `/app` folder.

[GitHub repo](https://github.com/payloadcms/payload) ·
[Official website](https://payloadcms.com) ·
[License: MIT](https://github.com/payloadcms/payload/blob/main/LICENSE)

## Overview

Payload is a headless CMS and backend framework where the entire application — data model, admin UI, access rules, and API surface — is described in a single TypeScript config. From that config it derives a React admin panel, a REST API, a GraphQL API, and a typed "Local API" for in-process database access, plus generated TypeScript types for every collection[^1]. The pitch is that you write your schema once and do not hand-author CRUD endpoints, admin forms, or migrations glue.

The defining event in Payload's history is the 3.0 rewrite (late 2024), which reoriented the project from a standalone Express application into a library that installs directly into an existing Next.js App Router project[^2]. In practice this means Payload's admin panel and routes are now Next.js route handlers and React Server Components living under your `/app` folder, and you can query the database directly inside your own server components via the Local API rather than round-tripping through HTTP. This is the project's biggest strength and its biggest constraint at once: it removes an entire deployment tier, but it couples Payload's supported path tightly to Next.js and its release cadence.

Payload is data-store-agnostic through database adapters. MongoDB (via Mongoose) is the historical default; Postgres and SQLite adapters built on Drizzle arrived later and are first-class but younger, and the relational adapters behave differently from Mongo for deeply nested, localized, and polymorphic fields[^3]. The company behind Payload was acquired by Figma in 2025; the project remains MIT-licensed and open source[^4].

## Getting Started

```bash
pnpx create-payload-app@latest
# or scaffold from the full-featured website template:
pnpx create-payload-app@latest -t website
```

```ts
// payload.config.ts — the entire app is described here
import { buildConfig } from 'payload'
import { mongooseAdapter } from '@payloadcms/db-mongodb'
import { lexicalEditor } from '@payloadcms/richtext-lexical'

export default buildConfig({
  secret: process.env.PAYLOAD_SECRET,
  db: mongooseAdapter({ url: process.env.DATABASE_URI }),
  editor: lexicalEditor(),
  collections: [
    {
      slug: 'posts',
      auth: false,
      access: { read: () => true },          // public read
      fields: [
        { name: 'title', type: 'text', required: true },
        { name: 'content', type: 'richText' },
        { name: 'author', type: 'relationship', relationTo: 'users' },
      ],
    },
    { slug: 'users', auth: true, fields: [] }, // auth-enabled collection
  ],
})
```

```ts
// Query directly in a React Server Component — no REST/GraphQL hop
import { getPayload } from 'payload'
import config from '@payload-config'

export default async function Page() {
  const payload = await getPayload({ config })
  const { docs } = await payload.find({ collection: 'posts', limit: 10 })
  return <ul>{docs.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

## Architecture / How It Works

Everything derives from `buildConfig`. Two top-level primitives structure the data model: **collections** (repeatable documents, e.g. `posts`, `users`) and **globals** (singletons, e.g. site settings). Each is a list of **fields**, and fields compose — `array`, `group`, `blocks`, `tabs`, and `relationship` nest arbitrarily. The `blocks` field is the layout-builder primitive: an ordered list of heterogeneous, schema-typed content blocks.

From that config Payload generates four access surfaces over the same documents: the **REST API** (`/api/{collection}`), the **GraphQL API** (schema + resolvers auto-built), the **Local API** (`payload.find/create/update/delete`, running in-process against the DB with no HTTP), and the **admin panel** (a React app, in v3 rendered through Next.js and extensible with your own server/client components). The Local API is what makes RSC data-fetching direct, and it is also how hooks and access functions call back into Payload without recursion through HTTP.

Cross-cutting behavior is expressed as **access control** functions and **hooks**. Access control is per-operation (`read`, `create`, `update`, `delete`) and can be enforced down to individual fields, returning booleans or query constraints. Hooks fire at document and field granularity (`beforeChange`, `afterRead`, `beforeValidate`, and so on) and are the primary extension mechanism. On top of these primitives Payload layers auth (JWT in HTTP-only cookies, with an auth-enabled collection standing in as the user model), **versions and drafts**, **localization** per field, and upload/media handling.

The **database adapter** abstracts persistence. `@payloadcms/db-mongodb` maps documents onto Mongoose; `@payloadcms/db-postgres` and `@payloadcms/db-sqlite` map onto Drizzle with generated relational schemas and SQL migrations. Rich text runs through a pluggable editor interface, now defaulting to **Lexical** (Meta's editor framework); the older Slate editor remains available as a package[^5].

## Production Notes

**v2 → v3 is a migration, not an upgrade.** v3 moved from Express to Next.js and changed the runtime model, admin bundling, and many config internals. Existing v2 apps do not carry over by changing a version number; there is an official migration guide, and the practical effort is real for non-trivial apps[^2].

**Relational adapters are younger than Mongo.** The Postgres/SQLite (Drizzle) adapters are supported but historically have had more edge cases than MongoDB for localized fields, deeply nested `blocks`/`array` structures, and polymorphic relationships, and they require running SQL migrations that Mongo's schemaless store does not. If your data shape is heavily nested and localized, prototype it on your target database early rather than assuming parity[^3].

**Serverless has sharp edges.** Payload deploys serverlessly (Vercel, Cloudflare Workers are advertised one-click paths), but the config is loaded per cold start, connection pooling against Postgres needs a pooler (Neon/PgBouncer), and file uploads must go to object storage (S3/R2/Blob) rather than the local filesystem. The "deploy anywhere" claim is true but each target has its own storage/database wiring to get right.

**Config load and admin bundle cost.** Because the whole app is one config evaluated at startup and build, very large schemas (many collections, deeply composed fields) increase cold-start and build time, and the admin panel is a client bundle whose size grows with custom components. Type generation (`payload generate:types`) must be re-run when the config changes or your generated types drift silently.

**Version coupling.** Since the supported path is Next.js-native, Payload's peer dependencies pin it to specific Next.js and React major versions; upgrading one can force upgrading the others, and third-party admin components can lag those ranges.

## When to Use / When Not

**Use when:**
- You are already building in Next.js App Router and want the backend/CMS in the same repo and `/app` folder.
- You want a TypeScript-first, self-hosted, no-vendor-lock-in CMS with generated types, granular access control, and a customizable admin.
- Your content model is genuinely structured (relationships, blocks, versions, localization) rather than a handful of flat forms.

**Avoid when:**
- You are not on Next.js and do not want to adopt it — v3's supported surface assumes it.
- You want a fully managed SaaS CMS with zero backend to operate (Payload is self-hosted infrastructure you own).
- You need a mature, battle-tested relational story today and your data is heavily nested/localized on Postgres — validate the adapter against your shape first.
- You want a stable, slow-moving dependency; Payload iterates quickly and just went through a full architectural rewrite.

## Alternatives

- strapi/strapi — established Node headless CMS with a plugin marketplace; use it if you want a standalone admin app decoupled from your frontend framework.
- directus/directus — wraps an existing SQL database and exposes it as an API/admin; use it when the database is the source of truth and already exists.
- keystonejs/keystone — also TypeScript, config-driven, GraphQL-first; use it when you want a similar code-as-schema model without the Next.js coupling.
- sanity-io/sanity — hosted content lake with a customizable Studio; use it when you want managed real-time collaboration and are fine with a SaaS backend.
- medusajs/medusa — use it instead when the core need is commerce rather than general content management.

## History

| Version | Date | Notes |
|---------|------|-------|
| Repo created | 2021-01 | Initial development, Express-based, MongoDB-only[^6]. |
| 1.0 | 2022 | First stable release; config-driven CMS with REST/GraphQL/admin[^1]. |
| 2.0 | 2023 | Lexical rich text, live preview, and other additions[^5]. |
| 3.0 | 2024 | Next.js-native rewrite; installs into `/app`, RSC + Local API, Postgres/SQLite adapters[^2]. |
| — | 2025 | Payload acquired by Figma; remains MIT open source[^4]. |

## References

[^1]: Payload docs, "What is Payload". https://payloadcms.com/docs/getting-started/what-is-payload
[^2]: Payload 3.0 migration guide. https://github.com/payloadcms/payload/blob/main/docs/migration-guide/overview.mdx
[^3]: Payload database adapters overview. https://payloadcms.com/docs/database/overview
[^4]: Repository description and license metadata (MIT), payloadcms/payload, retrieved 2026-07-15. https://github.com/payloadcms/payload
[^5]: Payload rich text (Lexical) docs. https://payloadcms.com/docs/rich-text/overview
[^6]: GitHub repository metadata: created 2021-01-05, 43.5k stars, 3.9k forks, last pushed 2026-07-14. https://github.com/payloadcms/payload

## Tags

typescript, headless-cms, cms, nextjs, nodejs, graphql, mongodb, postgres, react, content-management, backend-framework, mit-license
