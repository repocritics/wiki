# keystonejs/keystone

> A schema-driven headless CMS and Node.js app framework: describe your data model in TypeScript, get a GraphQL API and an auto-generated Admin UI.

[GitHub repo](https://github.com/keystonejs/keystone) ·
[Official website](https://keystonejs.com) ·
[License: MIT](https://github.com/keystonejs/keystone/blob/main/LICENSE)

## Overview

Keystone is a headless CMS built as a Node.js framework rather than a hosted product. You write a schema — lists of fields, in TypeScript — and Keystone generates a GraphQL API, a Prisma-backed database layer, and a React Admin UI to edit the data[^1]. It is developed and maintained by Thinkmill, an Australian agency, and the current line ships under the `@keystone-6/*` npm namespace[^2]. As of 2026 the repo is actively maintained (pushed to within the last day of this writing) but development cadence is agency-paced, not foundation- or VC-scale — releases are steady rather than frequent.

The defining tension is "framework, not product." Unlike Contentful or Sanity, Keystone gives you no hosting, no CDN, and no editing SaaS; you own the deployment, the database, and the runtime. In exchange you get a schema-as-code model where access control, hooks, and business logic are plain functions colocated with your data definitions. This suits teams who want a bespoke back-end without building the GraphQL + admin plumbing themselves, and frustrates teams expecting a turnkey CMS.

The second tension is lineage. "Keystone" has meant three substantially different codebases over its life — the original Express/MongoDB CMS (v4 and earlier), Keystone 5 (a GraphQL rewrite), and Keystone 6 (another rewrite on Prisma)[^3]. None migrate cleanly to the next. Reading old tutorials or Stack Overflow answers without checking which major version they target is the most common source of confusion.

## Getting Started

```bash
npm init keystone-app@latest my-app
cd my-app
npm run dev          # Admin UI on http://localhost:3000, GraphQL at /api/graphql
```

```ts
// keystone.ts
import { config, list } from '@keystone-6/core'
import { text, relationship, timestamp } from '@keystone-6/core/fields'
import { allowAll } from '@keystone-6/core/access'

export default config({
  db: { provider: 'sqlite', url: 'file:./keystone.db' },
  lists: {
    Post: list({
      access: allowAll,
      fields: {
        title: text({ validation: { isRequired: true } }),
        content: text({ ui: { displayMode: 'textarea' } }),
        author: relationship({ ref: 'User.posts', many: false }),
        createdAt: timestamp({ defaultValue: { kind: 'now' } }),
      },
    }),
    User: list({
      access: allowAll,
      fields: {
        name: text(),
        posts: relationship({ ref: 'Post.author', many: true }),
      },
    }),
  },
})
```

The schema above produces CRUD GraphQL operations, TypeScript types, a Prisma schema, and an editable Admin UI with no further code.

## Architecture / How It Works

Keystone composes several well-known pieces rather than reinventing them:

- **Schema layer** — Your `lists` config is the single source of truth. Field types (`text`, `relationship`, `select`, `image`, `document`, etc.) are functions that describe database columns, GraphQL types, validation, and their Admin UI rendering simultaneously.
- **Database** — Keystone 6 delegates persistence to **Prisma**. Keystone generates a `schema.prisma` from your lists and runs Prisma Migrate; supported providers are SQLite, PostgreSQL, and MySQL[^4]. You do not write Prisma models directly — Keystone owns that file.
- **API** — A GraphQL server (historically Apollo Server, mounted on an Express app) exposes queries and mutations derived from the schema. There is no REST surface. Inside hooks and access-control functions you use a server-side query API (`context.query`, `context.db`) rather than issuing HTTP calls.
- **Admin UI** — A React application (bundled with Next.js internally) that renders forms from the same field definitions. Each field type ships a "view" — a React component — so customizing how a field looks means writing a custom view.
- **Access control & hooks** — Per-list and per-field `access` functions and `hooks` (`resolveInput`, `validateInput`, `beforeOperation`, `afterOperation`) are plain async functions that run per request. This is where authorization and side effects live; there is no separate policy DSL.

The `document` field is a distinctive piece: a structured, JSON-based rich-text field with a configurable schema for inline components, rather than an opaque HTML blob[^5]. It is powerful but is Keystone-specific, which couples your content shape to Keystone's editor.

## Production Notes

- **Prisma migrations are the operational reality.** In development Keystone can auto-push schema changes; in production you commit generated migrations and run them explicitly (`db.useMigrations`). Schema drift between environments surfaces as Prisma migration errors, not Keystone errors, so operators need to understand Prisma's migration model, not just Keystone's.
- **The Admin UI is not a headless editing API.** It is a coupled React/Next app served by Keystone. You cannot easily swap it for a separate front-end deployment, and deep customization beyond field views (custom pages, branding, navigation) is limited and occasionally requires reaching into internal APIs that shift between minor versions.
- **Single-process coupling.** The GraphQL API and Admin UI are served from the same Keystone process by default. Scaling the API horizontally also scales the admin app; teams wanting them separated end up running the API-only mode and hosting a front-end themselves.
- **No first-party managed hosting.** You run it on your own Node infrastructure (a container, a VM, or a Node-compatible platform). Serverless deployment is awkward because Keystone expects a long-lived process and a persistent database connection.
- **Version lineage is a migration footgun.** There is no supported automated path from Keystone 5 to Keystone 6; the field APIs, access model, and database layer all changed. Keystone 5 lives in a separate repository in maintenance mode[^6].
- **TypeScript inference cost.** Large schemas generate large generated type files; editor responsiveness and build times degrade on very large list counts, a common complaint on big installations.

## When to Use / When Not

**Use when:**
- You want a code-defined content model with a GraphQL API and a working admin UI out of the box.
- Your team is comfortable owning deployment, database, and upgrades.
- You need access control and business logic expressed as TypeScript, colocated with the schema.
- You are building a bespoke back-end where a SaaS CMS would be too rigid.

**Avoid when:**
- You want turnkey hosted editing (Contentful, Sanity, or Strapi Cloud fit better).
- Non-technical editors need a heavily branded, deeply customized admin experience.
- You need REST, or a schema editable by non-developers at runtime — Keystone's schema is code and requires a redeploy.
- You are targeting serverless-only infrastructure.

## Alternatives

- strapi/strapi — use instead when you want an admin-panel-first CMS with plugins and a hosted-cloud option, and less code-as-schema.
- directus/directus — use instead when you want to wrap an existing SQL database and expose it, rather than have the CMS own the schema.
- payloadcms/payload — use instead when you want a similar TypeScript, code-first headless CMS with a more actively marketed ecosystem and native Next.js integration.
- prisma/prisma — use instead when you only need the ORM/migration layer and will build the API and admin yourself.
- hasura/graphql-engine — use instead when you want instant GraphQL over Postgres without a Node application framework or admin UI opinions.

## History

| Version | Date | Notes |
|---------|------|-------|
| Keystone 4 | ~2013–2018 | Original CMS on Express + MongoDB (Mongoose)[^3]. |
| Keystone 5 | 2019 | GraphQL rewrite with pluggable adapters; now in maintenance mode[^6]. |
| Keystone Next | 2020 | Prerelease of the next generation; later renamed Keystone 6. |
| Keystone 6 GA | 2021-08 | General availability on Prisma + `@keystone-6/*` packages[^7]. |
| Ongoing | 2022–2026 | Iterative `@keystone-6/core` releases; steady maintenance by Thinkmill. |

## References

[^1]: Keystone documentation, "Why Keystone." https://keystonejs.com/why-keystone
[^2]: Keystone README — packages published under `@keystone-6/*`, a Thinkmill project. https://github.com/keystonejs/keystone
[^3]: Keystone project history / "Keystone 5 and beyond." https://github.com/keystonejs/keystone-5/issues/21
[^4]: Keystone docs, "Database" (Prisma providers: SQLite, PostgreSQL, MySQL). https://keystonejs.com/docs/config/config#db
[^5]: Keystone docs, "Document field." https://keystonejs.com/docs/guides/document-fields
[^6]: keystonejs/keystone-5 — maintenance-mode repository for Keystone 5. https://github.com/keystonejs/keystone-5
[^7]: Keystone updates / release notes. https://keystonejs.com/updates

## Tags

typescript, headless-cms, graphql, nodejs, prisma, react, cms-framework, admin-ui, schema-first, backend
