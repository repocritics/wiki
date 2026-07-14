# medusajs/medusa

> Headless, MIT-licensed commerce platform for Node.js — a customizable framework of commerce modules rather than a turnkey store.

[GitHub repo](https://github.com/medusajs/medusa) ·
[Official website](https://medusajs.com) ·
[License: MIT](https://github.com/medusajs/medusa/blob/develop/LICENSE)

## Overview

Medusa is a headless commerce platform written in TypeScript, built by a Danish company of the same name (founders Sebastian Rindom and Kasper Kristensen)[^1]. It provides the server-side primitives of an online store — products, carts, orders, pricing, inventory, fulfillment, payments, promotions, tax — as open-source npm modules, and leaves the storefront entirely to you. There is no bundled shopping UI; Medusa ships a REST/admin API and a React admin dashboard, and you build the customer-facing site yourself (a Next.js starter storefront is provided separately).

The defining tension is **framework vs. product**. Medusa markets itself as "building blocks for digital commerce," and that is accurate: unlike Shopify (SaaS) or WooCommerce (a WordPress plugin), Medusa expects you to write code. The payoff is that core commerce logic — the parts that are genuinely hard and rarely differentiating — comes pre-built and can be extended without forking. The cost is that a Medusa store is a software project with a backend to host, upgrade, and operate, not a checkout you configure.

The second defining fact is the **v1 → v2 rewrite**. Medusa 2.0[^2] replaced the original monolithic Express-based backend with a modular architecture (isolated modules, a workflow engine, and module links). The two versions are close to different products; a v1 store does not "upgrade" to v2 so much as get rebuilt against it. Most content, plugins, and Stack Overflow answers written before late 2024 describe v1 and do not apply. Treat the version boundary as the single most important thing to get right before reading anything about Medusa.

## Getting Started

```bash
npx create-medusa-app@latest my-store
# scaffolds the Medusa backend + admin, and optionally a Next.js storefront
cd my-store
npm run dev            # admin at http://localhost:9000/app
```

A minimal custom module (v2), the unit of extension in Medusa:

```ts
// src/modules/hello/service.ts
import { MedusaService } from "@medusajs/framework/utils";
import { Model } from "@medusajs/framework/utils";

const Hello = Model.define("hello", {
  id: Model.id().primaryKey(),
  name: Model.text(),
});

class HelloModuleService extends MedusaService({ Hello }) {}
export default HelloModuleService;
```

```ts
// src/modules/hello/index.ts
import { Module } from "@medusajs/framework/utils";
import HelloModuleService from "./service";

export default Module("hello", { service: HelloModuleService });
```

The service auto-generates CRUD methods (`createHellos`, `listHellos`, …) from the model. Modules are registered in `medusa-config.ts` and wired to HTTP routes and workflows separately.

## Architecture / How It Works

Medusa 2.0 is organized around four concepts[^3]:

1. **Modules** — Self-contained packages that own a data model and a service. Each module is isolated: it cannot import another module's service directly, and it typically owns its own tables. Core commerce (product, pricing, inventory, order, cart, payment, fulfillment, tax, promotion, etc.) ships as first-party modules. Your custom logic is also a module. Persistence is via MikroORM against PostgreSQL.

2. **Workflows** — Durable, step-based orchestration. A workflow is a sequence of steps, each with an optional compensation function (rollback). The workflow engine persists state, so a long-running or failing workflow can retry and roll back partial work. Core operations like "complete cart" are themselves workflows you can hook into or replace.

3. **Module Links** — Because modules cannot hold foreign keys into each other, associations across module boundaries are declared as *links* in a separate link module. A product-to-price relationship, for example, is a link, not a column. This is what keeps modules swappable, and it is the concept most new users trip on.

4. **Query** — A read layer ("remote query" / the Query graph) that fetches and joins data across modules and their links in one call, hiding the fact that the data lives in isolated services.

The admin dashboard is a React application served by the backend. The HTTP layer exposes store and admin REST APIs; you add custom endpoints as API route files. Redis is used (optionally but strongly recommended in production) for the event bus, workflow engine, and caching — the defaults are in-memory and single-process only.

The whole design optimizes for *replaceability*: swap the payment module, override a workflow step, extend a model, without patching core. The price is indirection. Simple changes ("add a field and show it in admin") touch a model, a migration, a link or API extension, and admin customization — more moving parts than a monolith would demand.

## Production Notes

**Redis is not optional at scale.** The default event bus, cache, and workflow engine run in-memory, which means single-instance only and no durability. Production deployments must configure the Redis-backed modules; otherwise events are lost on restart and you cannot run more than one backend instance.

**Postgres + migrations.** Medusa depends on PostgreSQL and MikroORM migrations. Custom modules require you to generate and run migrations for their tables; forgetting this is a common "works locally, breaks on deploy" failure. There is no MySQL/SQLite production path.

**The v1 → v2 migration is a rebuild.** Data can be migrated with effort, but plugins, custom code, API shapes, and admin extensions were rewritten. Budget a v2 adoption as a project, and do not assume a v1 plugin has a v2 equivalent — verify each dependency first.

**Storefront is your problem.** Medusa gives you an API, not a store. Checkout UX, SEO, caching, and payment-provider client integration are all yours to build and own. The official Next.js starter is a starting point, not a finished storefront.

**Node/backend hosting.** Because it is a stateful Node backend with Postgres and Redis, Medusa does not fit "static + serverless" hosting models cleanly. It runs on a long-lived Node process (containers, VMs, or the vendor's managed Medusa Cloud[^4]). Serverless cold starts and the workflow engine's long-running steps are in tension.

**Moving target.** v2 iterates quickly. Minor releases regularly add modules and adjust framework APIs; pin versions and read release notes before upgrading, especially for anything touching workflows, links, or the module SDK.

## When to Use / When Not

**Use when:**
- You need a customizable commerce backend and have (or want) an engineering team to own it.
- Your model deviates from stock retail: B2B, marketplaces, subscriptions, distributor/PoS, service commerce.
- You want to avoid SaaS lock-in and per-transaction fees, and keep data and logic in your own infra.
- You're already a TypeScript/Node shop and want to extend commerce in the same language.

**Avoid when:**
- You want a store live this week with no backend to operate — Shopify or WooCommerce will be faster.
- You have no backend engineering capacity; Medusa is a codebase, not a dashboard.
- Your requirements are plain-vanilla retail with no custom logic — the framework's flexibility is overhead you won't use.
- You need a GraphQL-first API out of the box (Medusa's public API is REST; its cross-module Query graph is internal).

## Alternatives

- vendure-ecommerce/vendure — TypeScript/NestJS headless commerce, GraphQL-first; use instead when you want a GraphQL API and a NestJS-shaped codebase.
- saleor/saleor — Python/Django GraphQL commerce; use when your stack is Python or you want a more batteries-included GraphQL API.
- spree/spree — Ruby on Rails commerce engine; use in Rails shops with existing Ruby expertise.
- woocommerce/woocommerce — WordPress plugin; use for content-driven, smaller stores where WordPress is already the CMS.
- bagisto/bagisto — Laravel/PHP commerce; use when the team lives in the PHP/Laravel ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-01 | Public repo opened; v1 monolithic Express backend[^1]. |
| 1.x | 2021–2024 | Plugin ecosystem, admin dashboard, payment/fulfillment providers. |
| 2.0 | 2024-10 | Ground-up rewrite: modules, workflow engine, module links, Query[^2]. |

## References

[^1]: Medusa — company and project background. https://medusajs.com/
[^2]: Medusa 2.0 announcement — "The commerce framework." https://medusajs.com/blog/
[^3]: Medusa architecture and framework docs. https://docs.medusajs.com/learn
[^4]: Medusa Cloud (managed hosting). https://medusajs.com/cloud/

## Tags

typescript, nodejs, ecommerce, commerce, headless, framework, postgresql, backend, mit-license, react
