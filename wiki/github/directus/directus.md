# directus/directus

> A backend that wraps any existing SQL database with instant REST + GraphQL APIs and a Vue admin studio — now under a source-available license, not OSI open source.

[GitHub repo](https://github.com/directus/directus) ·
[Official website](https://directus.com) ·
License: MSCL 1.0 (source-available; SPDX: NOASSERTION)

## Overview

Directus is a Node.js/TypeScript application that connects to an existing SQL database, introspects its schema, and exposes that schema as auto-generated REST and GraphQL APIs plus a Vue 3 admin interface it calls the Studio[^1]. Its defining stance is "database-first": Directus does not own an opaque data model the way most CMSes do. It mirrors *your* tables and columns, so the same database is usable directly by SQL, by other applications, and by Directus simultaneously. This is what makes it usable as a headless CMS, an internal admin panel, or a general data-access layer over a database you already have.

The defining tension is the license. Directus was GPLv3 open source for most of its history, but has since relicensed to the **Monospace Sustainable Core License (MSCL) 1.0**, a source-available license derived from the Fair Core License[^2]. Per the project, it is free for organizations under $5M annual revenue and 50 employees (via an "Open Innovation Grant" key), with a commercial license required above those thresholds. This means the current code is *not* OSI-approved open source, despite the repo's history and framing. For a catalog of OSS, that distinction is the single most important thing to understand before adopting it.

The second thing to understand is the coupling to your database. Directus writes its own configuration, users, permissions, and metadata into `directus_*` system tables inside the same database it serves. Convenient, but it means Directus is now a stakeholder in your schema, and disentangling it later is real work.

## Getting Started

The fastest path is the CLI scaffolder against a fresh SQLite file, or Docker for anything real:

```bash
npm init directus-project@latest my-project
```

```bash
# Docker — Postgres assumed to exist and be reachable
docker run -p 8055:8055 \
  -e KEY=replace-with-random-value \
  -e SECRET=replace-with-random-value \
  -e DB_CLIENT=pg \
  -e DB_HOST=host.docker.internal -e DB_PORT=5432 \
  -e DB_DATABASE=mydb -e DB_USER=me -e DB_PASSWORD=secret \
  -e ADMIN_EMAIL=admin@example.com -e ADMIN_PASSWORD=changeme \
  directus/directus
```

Once running, the REST API is live immediately against whatever tables Directus finds:

```bash
# Auth, then read a collection — no route code was written
curl -X POST http://localhost:8055/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@example.com","password":"changeme"}'

curl 'http://localhost:8055/items/articles?fields=id,title&filter[status][_eq]=published' \
  -H 'Authorization: Bearer <access_token>'
```

## Architecture / How It Works

Directus is a pnpm monorepo. The core is a Node.js API server; the Studio is a separate Vue 3 single-page app that talks to that same API. Everything the Studio does is a public API call — there is no privileged admin backchannel.

- **Schema introspection.** On boot and on schema change, Directus reads the database's information schema and builds an in-memory representation of collections (tables), fields (columns), and relations. Database access goes through **Knex.js**, which is what lets one codebase target Postgres, MySQL, MariaDB, SQL Server, SQLite, OracleDB, CockroachDB, and Redshift[^1]. Directus stores its own metadata (field display config, interfaces, permissions) in `directus_*` tables alongside your data.
- **API generation.** REST endpoints (`/items/:collection`) and a GraphQL schema are derived from the introspected model. The REST query language (`filter`, `fields`, `deep`, `sort`, `aggregate`) is the primary surface and is more complete than the GraphQL layer in practice.
- **Permissions.** Access control is role- and (since v11) policy-based, resolvable down to the field level and expressible as row filters. AI/MCP agents run under the same policies as human users — there is no separate agent permission model[^3].
- **Flows.** An event- and schedule-driven automation engine built from operations (run script, webhook, condition, transform). This is Directus's answer to "no-code backend logic."
- **Extensions.** Hooks, endpoints, interfaces, displays, layouts, panels, modules, and operations are all pluggable via an Extensions SDK. This is how you escape the no-code ceiling.
- **Realtime.** WebSocket and GraphQL subscription support for live data, added in the v10 line.
- **Assets.** Pluggable file storage (local, S3, GCS, Azure) with on-the-fly image transformation and caching.

## Production Notes

**Redis is effectively mandatory at scale.** A single Directus node runs without it, but horizontal scaling does not: the cache, rate limiter, WebSocket coordination, and — critically — cross-node schema/permission cache invalidation all use Redis as the message bus. Run more than one replica without it and nodes will serve stale schema after a change.

**Boot and schema cache on large databases.** Introspecting a database with hundreds of tables and thousands of columns is not free; cold boot and schema reloads can be slow. The schema cache mitigates this but must be shared (Redis) across replicas, and stale cache after out-of-band schema changes is a recurring support theme.

**Schema-as-code is opt-in and easy to skip.** Because you can change schema by clicking in the Studio, teams drift toward untracked, environment-specific schemas. The intended discipline is `directus schema snapshot` / `directus schema apply` to move schema through environments in CI. Adopt it early; retrofitting it after divergence is painful.

**The `directus_*` tables couple you.** Since Directus writes its config into your database, a plain SQL backup captures both data and Directus state — good for consistency, awkward if you wanted your app DB and your CMS config decoupled. Some teams give Directus its own database and connect app databases as separate concerns.

**Permissions are powerful and a footgun.** Field-level, filter-based, per-policy rules interact in ways that are hard to reason about, and complex permission trees add query overhead. Test the permission matrix explicitly; "why can this role see that field" debugging is common.

**Upgrade pains.** The v9 rewrite (Node/TypeScript, Vue 3) was a clean break from the older PHP-era Directus and shares little with it. Within the modern line, the v11 permissions/policies rework changed how access is modeled and required migration attention. Migrations run against the DB (`directus database migrate:latest`) — pin versions and back up before upgrading, since schema migrations touch the same database that holds your data.

## When to Use / When Not

**Use when:**
- You already have a SQL database and want instant APIs plus an admin UI over it without writing route code.
- Non-technical teammates (or MCP agents) need governed, direct access to live data.
- You want to keep the database as the source of truth and portable to plain SQL, not locked in an opaque CMS model.
- Your organization is under the free-tier thresholds, so the licensing cost is zero.

**Avoid when:**
- You need OSI-approved open source or permissive licensing — Directus is now source-available, and the commercial thresholds may apply to you.
- You want code-first, version-controlled content models as the default (Directus's default is UI-driven schema).
- Your data lives outside SQL, or you need a document/graph-native store.
- You want a minimal, single-purpose API and don't want a full admin app, Flows engine, and permission system in the deployment.

## Alternatives

- strapi/strapi — Node headless CMS, but content types are code/config-defined and it owns its schema; use it when you want a CMS-first model rather than wrapping an existing database.
- supabase/supabase — Postgres-only BaaS (PostgREST + auth + realtime + storage); use it when you're Postgres-native and want auth/edge functions bundled over a Studio for arbitrary SQL engines.
- hasura/graphql-engine — GraphQL-first over Postgres and others with fine-grained authz; use it when GraphQL is the primary interface and you don't need an admin CMS.
- nocodb/nocodb — Airtable-style spreadsheet UI over an existing SQL database; use it when the end-user surface should be a grid, not a CMS.
- payloadcms/payload — TypeScript, code-first headless CMS that is Next.js-native and MIT-licensed; use it when config-as-code and a permissive license matter more than database-first introspection.

## History

| Version | Date | Notes |
|---------|------|-------|
| 6–8 | 2015–2020 | PHP-based Directus with a Vue frontend; the legacy line. |
| 9.0 | 2021-11 | Full rewrite in Node.js/TypeScript with Vue 3; modern Directus begins[^1]. |
| 10.0 | 2023 | Realtime (WebSockets / GraphQL subscriptions), platform maturation. |
| 11.0 | 2024 | Access policies rework — permissions modeled as reusable policies. |
| — | recent | Relicensed from GPLv3 to source-available (MSCL 1.0, ex-Fair Core License); native MCP server and AI Assistant added[^2][^3]. |

## References

[^1]: Directus README and documentation — introspection, Knex-backed database support, Studio. https://directus.com/docs
[^2]: Directus license — Monospace Sustainable Core License (MSCL) 1.0, derived from the Fair Core License. https://directus.com/license · https://fcl.dev
[^3]: Directus AI & MCP documentation — native MCP server, policy-governed agent access. https://directus.com/docs/guides/ai

## Tags

typescript, headless-cms, backend-as-a-service, rest-api, graphql, sql, database, no-code, vue, self-hosted, source-available, mcp
