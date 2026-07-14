# saleor/saleor

> GraphQL-native, API-only headless commerce backend built on Django — you bring the storefront.

[GitHub repo](https://github.com/saleor/saleor) ·
[Official website](https://saleor.io) ·
[License: BSD-3-Clause](https://github.com/saleor/saleor/blob/main/LICENSE)

## Overview

Saleor is a headless e-commerce backend that exposes its entire surface as a single GraphQL API. It began in 2013 as a Django-based storefront from Mirumee, a Polish software house, and for its first several years shipped as a conventional server-rendered Django store with the shop UI in the same codebase[^1]. The 2.x line (2019) added a GraphQL API alongside a React dashboard and storefront. The 3.0 rewrite (2021) removed the server-rendered storefront from core entirely and committed to an API-only model: GraphQL is the *only* way to read data, place orders, or configure the system[^2].

The defining tension is the API-only bet itself. There is no plugin system in the WordPress/Magento sense — no PHP hooks, no theme layer, no in-process extension points. You extend Saleor from the outside via webhooks, apps (dashboard iframes with their own backends), metadata, and synchronous API extensions for payment/tax/shipping[^3]. This buys clean separation, independent deploys, and technology-agnostic extension, at the cost of operating several moving parts (API, workers, Redis, Postgres, a dashboard, and a storefront you write yourself) before you have a working store. Saleor's own docs are candid that a solo developer on a low-traffic shop may find this heavier than a Magento or WooCommerce quick-start[^1].

Saleor Commerce (formerly Mirumee) funds development and sells Saleor Cloud, a hosted version. The core stays single-edition open source under BSD-3-Clause — there is no feature-gated "enterprise" fork of the repo.

## Getting Started

The supported path is Docker Compose via the `saleor-platform` meta-repo, which wires the API, dashboard, Celery workers, Redis, and Postgres together[^4]:

```bash
git clone https://github.com/saleor/saleor-platform.git
cd saleor-platform
docker compose up
# API + GraphQL Playground at http://localhost:8000/graphql/
# Dashboard at http://localhost:9000/
```

A minimal query against a running instance — fetch products in a channel:

```graphql
query {
  products(first: 10, channel: "default-channel") {
    edges {
      node {
        name
        pricing {
          priceRange {
            start { gross { amount currency } }
          }
        }
      }
    }
  }
}
```

Everything — creating orders, managing catalog, configuring shipping — is a GraphQL mutation. There is no REST fallback and no admin HTML rendered by core.

## Architecture / How It Works

Saleor Core is a **Django** application. GraphQL is served through **Graphene** (`graphene-django`), Postgres is the required database, **Celery** handles asynchronous work (email, webhook delivery, exports, reindexing), and **Redis** backs Celery and caching.

Key structural pieces:

- **Channels** — the multichannel model introduced in 3.0. A channel scopes currency, pricing, stock allocation, and product availability. Most queries require a `channel` argument; the same catalog serves multiple storefronts (B2C, B2B, per-region) with independent pricing[^2].
- **GraphQL schema** — large, code-first via Graphene types. Because Django's ORM and GraphQL are a natural N+1 trap, Saleor leans heavily on **DataLoaders** to batch relational lookups. Much of the performance-critical code is dataloader wiring rather than resolver logic.
- **Permissions & auth** — JWT-based, with a granular permission enum. Apps authenticate with their own tokens and are granted scoped permissions.
- **Webhooks** — the primary extension mechanism. **Async** webhooks fire after events (order created, product updated) via Celery. **Sync** webhooks are called inline during a request and must return a response — these back payment authorization, shipping-method resolution, tax calculation, and checkout validation[^3].
- **Apps** — external services that register webhooks and optionally render a dashboard iframe. Saleor ships no first-party payment gateway logic in core beyond a dummy gateway; real payments (Stripe, Adyen, etc.) are apps that implement the transaction/payment webhooks.
- **Storefront & dashboard are separate repos** — `saleor-dashboard` (React) and `storefront` (Next.js App Router) are decoupled projects consuming the same API.

The result is a hub-and-spoke system: a stateless GraphQL core surrounded by independently deployed apps. The coupling that remains is to Postgres and to the GraphQL contract itself.

## Production Notes

**You are running a distributed system, not a shop.** A production deployment is, at minimum: the API (WSGI/ASGI), one or more Celery workers, Celery Beat, Redis, Postgres, a dashboard, plus every payment/tax/shipping app you depend on. Under-provisioning Celery workers silently delays webhook delivery and order-confirmation email.

**Sync webhooks are on the checkout critical path.** Payment, tax, and shipping-method resolution can be delegated to external apps that Saleor calls *synchronously* during checkout. If that app is slow or down, checkout latency degrades or fails. Budget for timeouts, retries, and app availability the same way you would a payment gateway.

**GraphQL N+1 is a standing hazard.** The API is expressive enough that a client can request deeply nested data (products → variants → attributes → media) and blow up query counts if a resolver lacks a dataloader. Watch for slow queries after schema or resolver changes; the mitigation is always a dataloader, not ORM `prefetch_related` sprinkled ad hoc.

**Postgres-only, and migration-heavy on large catalogs.** Saleor uses Postgres-specific features and does not target other databases. Minor-version upgrades within 3.x frequently ship data migrations; on large catalogs or order tables these can lock or run long, so stage upgrades against a production-sized dump.

**No storefront is provided.** The example `storefront` is a starting point, not a product. Building, theming, SEO, and performance of the customer-facing site are entirely your responsibility — the flip side of "unlimited SEO freedom."

**Schema size.** The GraphQL schema is large; introspection payloads and generated client types are big, and codegen build steps can be slow. Persisted queries and disciplined fragment usage help on the client side.

## When to Use / When Not

**Use when:**
- You want a headless commerce backend and are building a custom frontend (or several) against one API.
- You need genuine multichannel: multiple currencies, regions, or B2C/B2B storefronts from one catalog.
- Your team is comfortable operating Django + Celery + Postgres + Redis and deploying external services.
- You want a single open-source edition with no commercial feature gating in the code.

**Avoid when:**
- You want an out-of-the-box store with themes and one-click plugins — WooCommerce, Shopify, or Magento fit better.
- You are a solo developer on a low-traffic shop and the operational surface outweighs the flexibility (Saleor's own docs say so).
- You need a database other than Postgres, or want to avoid running background workers.
- You want in-process, language-native extension points rather than out-of-band webhooks and apps.

## Alternatives

- medusajs/medusa — Node.js/TypeScript headless commerce with an in-process module/plugin system; use when your team is JS-first and prefers extending in-process over webhooks.
- vendure-ecommerce/vendure — TypeScript/NestJS GraphQL headless commerce; use when you want a GraphQL API but a plugin architecture inside a single Node runtime.
- woocommerce/woocommerce — WordPress/PHP monolith; use when you want themes, a plugin marketplace, and a non-developer-friendly quick start over API-only purity.
- magento/magento2 — PHP enterprise monolith; use when you need a mature built-in feature set and are staffed for its operational weight.
- spree/spree — Ruby on Rails commerce; use when your stack is Rails and you want a conventional MVC store over a headless split.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013 | Open-sourced by Mirumee as a Django storefront[^1]. |
| 2.0 | 2019 | GraphQL API added; React dashboard and storefront introduced. |
| 3.0 | 2021 | API-only rewrite; storefront removed from core; channels / multichannel model[^2]. |
| 3.x | 2021–2026 | Frequent minor releases; the current production line for API, dashboard, and storefront. |

## References

[^1]: Saleor README and repository, open-sourced 2013 by Mirumee. https://github.com/saleor/saleor
[^2]: Saleor channels / multichannel documentation. https://docs.saleor.io/developer/channels/overview
[^3]: Saleor webhooks and app extensibility docs. https://docs.saleor.io/developer/extending/webhooks/overview
[^4]: Saleor Platform (Docker Compose meta-repo). https://github.com/saleor/saleor-platform

## Tags

python, django, graphql, headless-commerce, ecommerce, api-only, multichannel, webhooks, celery, postgresql, composable-commerce
