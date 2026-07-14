# teableio/teable

> An Airtable-style no-code database that runs on real Postgres, so the data stays queryable as a database rather than trapped behind a proprietary API.

[GitHub repo](https://github.com/teableio/teable) ·
[Official website](https://teable.ai) ·
License: AGPL-3.0 (apps/plugins) + MIT (packages) — GitHub reports `NOASSERTION`

## Overview

Teable is a spreadsheet-database hybrid in the Airtable / NocoDB / Baserow category: a grid UI on top of a relational store, with views (grid, form, kanban, gallery, calendar), formula fields, links between tables, filtering, grouping, and real-time collaboration. It first appeared in late 2022[^1] and is maintained by a company that also runs a hosted version at teable.ai. The pitch that separates it from most no-code tools is data ownership: the backing store is a standard PostgreSQL database you can point psql, a BI tool, or another application at directly, rather than an opaque store reachable only through the vendor's REST API.

The defining tension is open-core. The self-hostable Community Edition is genuinely open source (AGPL-3.0 for the apps and plugins, MIT for the shared packages), but a set of features — AI, the permission/authority matrix, automation, advanced admin — live in a proprietary Enterprise Edition that is not in this repository[^2]. This is not a bait-and-switch; the split is documented. But it means feature comparisons against the hosted product or against fully-open competitors (NocoDB, Baserow CE) can mislead if you assume everything on the marketing site ships in the repo.

The second tension is the "spreadsheet that scales" claim. Teable advertises millions of rows in a spreadsheet-like grid[^3]. That is real, but it is achieved with virtualized rendering, server-side pagination, and Postgres doing the heavy lifting — it is a database with a grid on top, not a spreadsheet engine. Treating it like Excel (giant client-side formulas, whole-sheet recalculation) will not match the mental model the architecture is built for.

## Getting Started

Self-host with Docker Compose (Postgres + the Teable app):

```sh
# Standalone example bundles Postgres + Teable
git clone https://github.com/teableio/teable
cd teable/dockers/examples/standalone/
docker-compose up -d
# app on http://localhost:3000
```

Local development is a pnpm monorepo; the backend boots the frontend for you:

```sh
corepack enable
pnpm install
make switch-db-mode          # provisions Postgres
cd apps/nestjs-backend
pnpm dev                     # starts NestJS + Next.js together
```

One-click deploy templates exist for Railway, Sealos, Zeabur, and others[^1]. For anything beyond a demo, self-host the Docker image against a managed Postgres.

## Architecture / How It Works

Teable is a pnpm/Turborepo monorepo with a clear license and dependency split[^1]:

- `apps/nextjs-app` — Next.js frontend (AGPL-3.0).
- `apps/nestjs-backend` — NestJS backend (AGPL-3.0).
- `packages/core` — shared types and interfaces (MIT).
- `packages/sdk` — SDK used by extensions and plugins (MIT).
- `packages/db-main-prisma` — Prisma schema, migrations, and client (MIT).
- `packages/ui-lib`, `common-i18n`, `eslint-config-bases` — UI and tooling (MIT).
- `plugins/` — first-party plugins (AGPL-3.0).

The data model is the interesting part. Every user table is a real Postgres table, and Teable maintains a metadata layer (field definitions, view configs, links) alongside it. Field types — single/multiple select, link, formula, rollup, attachment — are projected onto Postgres columns, and computed fields (formula, rollup, lookup) are evaluated server-side and materialized so the grid can page through them without recomputing client-side. This is what makes the "connect your own BI tool to the Postgres" story work: the tables are legible SQL, not a key-value blob.

Real-time collaboration is built on an operational-transformation layer (ShareDB) over WebSockets, so multiple users editing the same view converge without last-write-wins clobbering. Prisma is the ORM for the metadata and schema-migration layer.

The coupling to Postgres is deliberate and deep. Although `sqlite` appears in the repo topics and early versions could run against SQLite for local use, the product is positioned and described as "No-Code Postgres"[^4], and the scaling story assumes Postgres. Do not plan a production SQLite deployment.

## Production Notes

- **Postgres is the whole game.** Row-count claims, formula performance, and concurrent-editing behavior all rest on the database. Give it a properly-sized managed Postgres with real connection pooling; a starved database instance is the first thing to hit under load, not the Node app.
- **Open-core feature gaps.** AI, the authority/permission matrix, automation, and advanced admin are Enterprise features and are not in the AGPL repo[^2]. If your requirements include fine-grained row/field permissions or automations, confirm whether the CE ships them before committing — this is the most common source of "the docs said it could do X" surprises.
- **AGPL-3.0 obligations.** The app code is AGPL. If you modify Teable and expose it to users over a network, the license's network-use clause applies. For internal-only deployments this is usually fine, but legal review is warranted before offering a modified Teable as a service. The MIT-licensed `packages/*` (core, sdk, db-main-prisma) can be depended on without the AGPL reach — that split is intentional so integrators can build on the SDK.
- **Default branch is `develop`.** Track tagged releases, not the tip of `develop`, for anything you run in production; the moving branch carries in-flight work.
- **Migrations.** Schema changes to user tables and metadata are Prisma-managed. Back up Postgres before upgrades, and treat major version bumps as migration events, not drop-in image swaps.
- **Attachments.** Files live in object storage (S3-compatible) configured via environment; budget for that separately from the database.

## When to Use / When Not

**Use when:**
- You want an Airtable-style UI but need the data to stay in a real, directly-queryable Postgres you control.
- You are self-hosting and data residency / ownership is a hard requirement.
- Your team lives in spreadsheet-grid workflows but the row counts have outgrown actual spreadsheets.
- You want to build on top via the MIT-licensed SDK/plugins.

**Avoid when:**
- You need the AI, automation, or granular-permission features and cannot pay for Enterprise — they are not in the open repo.
- AGPL-3.0 conflicts with how you intend to redistribute or offer the app.
- You want a fully-open feature set with no proprietary tier (look at NocoDB or Baserow CE).
- You need a true spreadsheet calculation engine rather than a database with a grid front-end.

## Alternatives

- nocodb/nocodb — NocoDB: turns an existing SQL database (MySQL/Postgres/SQLite) into a grid; use instead when you want to wrap a database you already have.
- bram2w/baserow — Baserow: Airtable-alternative with a large open CE; use instead when you want a fully-open feature set and a plugin marketplace.
- apitable/apitable — APITable: another Airtable-style tool; use instead when you want its widget/dashboard model, noting maintenance has been uneven.
- directus/directus — Directus: wraps any SQL database as a headless CMS/data platform with a strong API; use instead when the API and content-management layer matter more than the spreadsheet grid.
- supabase/supabase — Supabase: Postgres plus auto-generated APIs, auth, and a table editor; use instead when you are building an app backend and the grid is secondary.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-11 | Repository created; early Airtable-alternative on Postgres[^1]. |
| 1.0 | 2024 | First 1.x line — self-hostable Community Edition, Docker deploy, core views and fields. |
| open-core | 2024→ | Enterprise Edition split introduced (AI, permissions, automation) alongside AGPL CE[^2]. |
| active | 2026-07 | ~21.5k stars, ~1.3k forks; actively developed on the `develop` branch[^3]. |

## References

[^1]: teableio/teable README and repository structure. https://github.com/teableio/teable
[^2]: Teable pricing / Enterprise Edition (AI, authority matrix, automation, advanced admin are EE features). https://teable.ai/pricing
[^3]: GitHub repository metadata (stars, forks, default branch, last push), retrieved 2026-07-15. https://github.com/teableio/teable
[^4]: Teable repository description, "The Next Gen Airtable Alternative: No-Code Postgres." https://github.com/teableio/teable

## Tags

typescript, no-code, database, postgres, airtable-alternative, spreadsheet, nestjs, nextjs, self-hosted, realtime, low-code, open-core
