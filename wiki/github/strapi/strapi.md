# strapi/strapi

Strapi — an open-source headless CMS. Auto-generates REST + GraphQL APIs over a content model you define in the admin UI.

## What it is

A TypeScript / Node.js headless CMS that lets non-coders manage content via an admin UI while exposing the content as REST + GraphQL APIs for developers to consume. Self-host or use Strapi Cloud. The "headless" framing means Strapi handles the content + API; you build the frontend with any framework (Next.js, Nuxt, mobile apps).

## Key features

- Visual content modeling in the admin UI.
- Auto-generated REST + GraphQL APIs per content type.
- Role-based access control + i18n.
- Webhook + plugin system.
- Self-host (SQLite, Postgres, MySQL) or Strapi Cloud.
- MIT-licensed (core); enterprise features behind paid plans.

## Tech stack

- TypeScript primary.
- Node.js + Koa.
- React-based admin UI.

## When to reach for it

- You're building a frontend (Next.js, Nuxt, mobile) and need a CMS for content editors.
- You want auto-API generation rather than hand-rolling endpoints.
- You're self-hosting for compliance / data-residency reasons.

## When *not* to reach for it

- You want vendor-managed-only — Contentful, Sanity offer that.
- Your content needs are static / Markdown-friendly — use a static site generator with Markdown files.

## Maturity signal

Actively maintained under Strapi Inc. 10+ years.

## Alternatives

- Directus — comparable headless CMS.
- Contentful, Sanity, Hygraph — commercial managed.
- Payload CMS — code-first alternative.

## Tags

typescript, headless-cms, nodejs, api, content-management, self-hosted, mit-license
