# sanity-io/sanity

> Sanity Studio — a configurable, React-based editing environment that is open source, backed by a proprietary hosted content store (Content Lake).

[GitHub repo](https://github.com/sanity-io/sanity) ·
[Official website](https://www.sanity.io) ·
[License: MIT](https://github.com/sanity-io/sanity/blob/main/LICENSE)

## Overview

Sanity is a headless CMS whose open-source surface is **Sanity Studio**: a single-page React application you configure in TypeScript and run yourself. The repository is the monorepo for that Studio and its surrounding toolchain (the `sanity` package, `@sanity/*` libraries, GROQ tooling, CLI). It has been developed since 2017 and rewritten substantially for v3 (2022), which moved configuration from a JSON file to code and shipped the Studio as an embeddable npm package[^1].

The defining tension of the project is that **the editor is open source but the backend is not**. Content is stored in Sanity's hosted **Content Lake**, a proprietary real-time document store queried with GROQ (Sanity's own query language) or GraphQL[^2]. You do not self-host the datastore; you pay Sanity for API traffic, bandwidth, and dataset capacity on a pay-as-you-go plan. The MIT license on this repo covers the client-side Studio and libraries, not the service that makes them useful. Teams evaluating Sanity as "open-source CMS" should read it as "open-source editor on a managed backend" — the lock-in is at the data layer, not the UI.

What you get for that tradeoff is genuinely strong: structured content modeling with per-field validation, real-time collaborative editing, a well-regarded Portable Text block editor, on-demand image transforms, and content versioning/history via the datastore. The schema is the source of truth and drives the entire editing UI.

## Getting Started

```bash
npm create sanity@latest
# or: yarn create sanity@latest / pnpm create sanity@latest
```

```ts
// sanity.config.ts — Studio configuration is code, not JSON
import {defineConfig} from 'sanity'
import {structureTool} from 'sanity/structure'

export default defineConfig({
  projectId: 'your-project-id',
  dataset: 'production',
  plugins: [structureTool()],
  schema: {
    types: [
      {
        name: 'post',
        type: 'document',
        fields: [
          {name: 'title', type: 'string', validation: (r) => r.required()},
          {name: 'body', type: 'array', of: [{type: 'block'}]}, // Portable Text
        ],
      },
    ],
  },
})
```

Reading content from a frontend uses `@sanity/client` and a GROQ query:

```ts
import {createClient} from '@sanity/client'
const client = createClient({projectId, dataset, apiVersion: '2024-01-01', useCdn: true})
const posts = await client.fetch(`*[_type == "post"]{title, body}`)
```

## Architecture / How It Works

There are three layers, and only the first two live in this repo:

1. **Studio** — a React SPA. Your `sanity.config.ts` registers schema types, plugins, and "tools" (the panes down the left rail). The schema drives form generation: every field type maps to an input component, which you can override. Studio can run standalone (`sanity dev`) or be mounted inside another React app (e.g. embedded in a Next.js route).
2. **Client libraries + GROQ** — `@sanity/client`, the GROQ query engine, Portable Text serializers, image-URL builder. GROQ is a projection-based query language (not SQL, not GraphQL) designed for shaping nested documents in a single round-trip[^2]. A GraphQL API is also generated from the schema, but GROQ is the native and more expressive path.
3. **Content Lake** (not in this repo, not open source) — the hosted document store. It provides real-time listeners, full document history, and the asset pipeline. This is the paid service.

Schema is the coupling point. Because the editing UI, validation, and (optionally) the GraphQL API are all generated from the same schema definition, changing a field ripples through the Studio automatically — but there is no enforced migration story: the datastore is schemaless JSON, so removing or renaming a field does not migrate existing documents. You reconcile drift yourself with scripts against the API.

Portable Text is Sanity's answer to "rich text as data": rich content is stored as an array of typed blocks rather than HTML, so it can be re-serialized to any target (HTML, Markdown, native mobile, voice)[^3]. This is one of the more portable, standards-minded parts of the stack and is usable independent of Sanity's backend.

## Production Notes

- **Vendor lock-in is real and at the data layer.** Migrating off Sanity means exporting documents (`sanity dataset export`) and rebuilding queries — GROQ has no portable equivalent, so every query is rewritten. The Studio being MIT does not reduce this cost.
- **Billing is usage-based on API requests, bandwidth, and CDN.** Costs are predictable for editorial workloads but can surprise high-traffic sites that query the API directly instead of caching. Use `useCdn: true` and cache aggressively; treat the API as an origin, not a per-request datastore.
- **`useCdn` returns eventually-consistent data.** The CDN endpoint can lag the live dataset by a short window. For preview/draft flows you must hit the live API (`useCdn: false`) with a token, which is uncached and rate-limited differently.
- **GROQ has a learning curve.** It is powerful but idiosyncratic; joins (`->` dereferencing) and projections behave unlike SQL/GraphQL, and query performance depends on how documents are structured. There is no schema-enforced index tuning exposed to you.
- **Schema changes don't migrate data.** Renaming a field leaves old documents untouched. Plan migration scripts as a first-class task, not an afterthought.
- **v2 → v3 was a hard migration.** Config moved from `sanity.json` to `sanity.config.ts`, package names consolidated into `sanity`, and many plugins needed rewrites. Studios pinned to v2 predate this and are effectively legacy[^1].
- **Draft/publish and preview** require wiring perspective tokens and (for frameworks like Next.js) draft-mode plumbing; it is not zero-config.

## When to Use / When Not

**Use when:**
- You want structured, schema-driven content with a customizable editing UI and are fine with a hosted backend.
- Real-time collaboration, document history, and a strong block editor matter to your editorial team.
- You need rich text as portable data (Portable Text) that outlives any single frontend.
- You want to embed a CMS UI directly inside a React application.

**Avoid when:**
- You require a fully self-hosted, open-source data layer — the datastore is proprietary SaaS.
- You want to avoid a bespoke query language; GraphQL-first or SQL-backed CMSs fit better.
- Your content model is simple and flat; the schema/GROQ overhead may exceed the benefit.
- Predictable flat-rate pricing is a hard requirement and traffic is high and cache-hostile.

## Alternatives

- strapi/strapi — self-hostable Node CMS with your own database; choose when you must own the backend and want REST/GraphQL out of the box.
- directus/directus — wraps an existing SQL database as a headless CMS; choose when data already lives in Postgres/MySQL and SQL access matters.
- payloadcms/payload — code-first, self-hosted TypeScript CMS with its own admin UI; choose when you want Sanity-like config-as-code without a proprietary datastore.
- tinacms/tinacms — Git-backed editing on top of Markdown/MDX; choose when content should live in your repo, not a service.
- contentful/contentful (SaaS, not OSS) — choose when you want a managed CMS with flat enterprise contracts rather than usage-based billing.

## History

| Version | Date | Notes |
|---------|------|-------|
| Early releases | 2017–2019 | `@sanity/*` packages, `sanity.json` config, Content Lake + GROQ. |
| v2 | 2020 | Consolidated v2 line; JSON-configured Studio, plugin ecosystem matures. |
| v3 | 2022-12 | Major rewrite: config-as-code (`sanity.config.ts`), unified `sanity` package, React 18, embeddable Studio[^1]. |
| v4 | 2025 | Latest major line; continued Studio and toolchain evolution. |

## References

[^1]: Sanity blog, "Sanity Studio v3" — the config-as-code rewrite. https://www.sanity.io/blog/sanity-studio-v3
[^2]: GROQ — Graph-Relational Object Queries, Sanity's query language. https://www.sanity.io/docs/how-queries-work
[^3]: Portable Text — rich text as structured data. https://www.sanity.io/docs/presenting-block-text

## Tags

typescript, cms, headless-cms, react, structured-content, groq, portable-text, realtime, saas-backed, content-management
