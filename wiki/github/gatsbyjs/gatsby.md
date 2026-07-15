# gatsbyjs/gatsby

> React static-site framework built around a GraphQL data layer — historically important, now in low-activity maintenance under Netlify.

[GitHub repo](https://github.com/gatsbyjs/gatsby) ·
[Official website](https://www.gatsbyjs.com) ·
[License: MIT](https://github.com/gatsbyjs/gatsby/blob/master/LICENSE)

## Overview

Gatsby is a React framework for building content-driven websites, created by Kyle Mathews with a first stable release in 2017[^1]. Its defining idea was to pull data from arbitrary sources — Markdown files, headless CMSes like Contentful or WordPress, REST/GraphQL APIs — into a single unified GraphQL layer, then statically render React pages against it at build time. For several years around 2018–2021 it was the default choice for React-based blogs, marketing sites, and documentation, and its plugin/source ecosystem was its main draw.

The framework's central tension is that same GraphQL data layer. It gives you one consistent query interface regardless of backend, but it also means that even reading a local Markdown file goes through a schema-inference and query-resolution pipeline. That pipeline is the source of both Gatsby's flexibility and its most-cited pain: slow builds, opaque schema errors, and a learning curve that has little to do with React itself.

Momentum matters here more than for most entries in this catalog. Gatsby Inc. was acquired by Netlify in February 2022[^2], after which the core team was substantially reduced and the associated Gatsby Cloud hosting product was wound down and folded into Netlify[^3]. The last major version (v5) shipped in November 2022; commits since then have been dominated by dependency maintenance rather than feature work. Treat Gatsby as a stable, still-functional but effectively feature-frozen framework, and weigh new adoption against that trajectory.

## Getting Started

```shell
npm init gatsby
# scaffolds a site; then:
cd my-gatsby-site
npm run develop      # dev server at http://localhost:8000
```

```jsx
// src/pages/index.js — a page component with a GraphQL page query
import * as React from "react"
import { graphql } from "gatsby"

export default function Home({ data }) {
  return <h1>{data.site.siteMetadata.title}</h1>
}

// Data is resolved at build time from the internal GraphQL store
export const query = graphql`
  query {
    site { siteMetadata { title } }
  }
`
```

The GraphiQL explorer at `http://localhost:8000/___graphql` is the practical way to discover what the schema actually contains — the schema is inferred from your data, so it is rarely what you would hand-write.

## Architecture / How It Works

A Gatsby build runs in phases: **source** (source plugins ingest data as "nodes"), **transform** (transformer plugins convert nodes, e.g. Markdown → HTML), **schema inference** (a GraphQL schema is generated from the shape of all nodes), **query resolution** (page and static queries are run against the store), and **render** (React pages are rendered to static HTML, then hydrated in the browser).

- **The GraphQL store is the hub.** Everything — filesystem, CMS, APIs — is normalized into nodes and queried uniformly. This is the feature and the coupling: you cannot meaningfully use Gatsby without adopting its data layer, and much ecosystem knowledge is about plugin configuration rather than React.
- **Plugins do most of the work.** `gatsby-source-*` ingest data, `gatsby-transformer-*` reshape it, and general plugins (`gatsby-plugin-image`, `gatsby-plugin-sharp`) handle cross-cutting concerns. The registry is large but unevenly maintained; many popular plugins have not had substantive updates since the acquisition.
- **Rendering options.** Originally pure Static Site Generation (SSG). Gatsby v4 added Deferred Static Generation (DSG, render on first request) and Server-Side Rendering (SSR) via `getServerData`, selectable per page. These reduce build time for large sites but require a Node runtime at serve time, undercutting the "pure CDN" story.
- **Image handling** via `gatsby-plugin-image` + `sharp` is one of the genuinely strong pieces: responsive images, blur-up placeholders, and format negotiation generated at build time.

The monorepo is managed with Lerna and publishes dozens of separate npm packages (`gatsby`, `gatsby-cli`, the plugins) from one repository[^1].

## Production Notes

**Build time is the headline operational concern.** Because every page query runs against the inferred GraphQL store, build cost scales with content volume and query complexity, not just page count. Large sites (thousands of pages, image-heavy) can see multi-minute to multi-tens-of-minutes builds. Mitigations: incremental builds, DSG for rarely-visited pages, and constraining image processing.

- **`sharp` image processing is CPU- and memory-hungry.** On constrained CI runners it is a common cause of OOM kills and long builds; parallelism and the number of generated image variants are the levers.
- **Schema inference is fragile.** If a field is present on some nodes and absent on others, inferred types can shift between builds, producing errors that only appear when content changes. Explicit schema customization (`createTypes`) avoids this but is extra work.
- **Hosting assumptions changed.** SSG output is a static bundle any CDN can serve, but DSG/SSR require a Node function host. With Gatsby Cloud gone, the supported path is Netlify's Gatsby adapter; other hosts work for pure SSG but have less first-party support[^3].
- **Upgrade friction.** Each major (v2→v3→v4→v5) raised the minimum Node version and shifted plugin APIs; v3 introduced fast refresh and query-on-demand in dev, v4 the DSG/SSR engine. Plugin lag around each bump was a recurring problem, and is worse now that many plugins are unmaintained.
- **Maintenance risk.** Security and dependency updates still land, but do not plan around new features or fast turnaround on issues. Open issues sit long; the `pushed_at` timestamp reflects dependency housekeeping, not active development.

## When to Use / When Not

**Use when:**
- You have an existing Gatsby site that works — it remains stable and there is no forced-migration pressure.
- You want a React SSG with a mature image pipeline and a specific source plugin (e.g. an established CMS integration) that already exists and is maintained.
- Your content set is modest and build times stay comfortable.

**Avoid when:**
- You are starting a new project and want an actively developed framework — the ecosystem has moved to Next.js and Astro.
- Build performance at scale is a hard requirement and you don't want to manage DSG/incremental builds.
- You want to avoid learning a GraphQL data layer just to render local Markdown.
- You need long-term vendor commitment to new features rather than maintenance.

## Alternatives

- withastro/astro — content-site-first with islands architecture; the most common modern replacement for Gatsby's blog/docs use case, leaner builds, no mandatory GraphQL layer.
- vercel/next.js — use when you want an actively developed React framework covering SSG, SSR, ISR, and server components in one tool.
- 11ty/eleventy — use when you want a simple, fast static generator without React or a data-layer abstraction.
- sveltejs/kit — use when you want a comparable content/app framework without React.
- nuxt/nuxt — use when your team is on Vue rather than React.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | First stable release; GraphQL data layer, plugin model[^1]. |
| 2.0 | 2018-09 | Faster builds, schema improvements, React 16. |
| 3.0 | 2021-03 | Incremental builds, query-on-demand, fast refresh in dev. |
| 4.0 | 2021-09 | Deferred Static Generation (DSG) and SSR via `getServerData`. |
| — | 2022-02 | Gatsby Inc. acquired by Netlify[^2]. |
| 5.0 | 2022-11 | Slices API, partial hydration (beta), Node 18, GraphiQL v2. |

## References

[^1]: Gatsby repository README and packages tree — monorepo managed with Lerna. https://github.com/gatsbyjs/gatsby
[^2]: Netlify, "Netlify acquires Gatsby" — announced 2022-02-01. https://www.netlify.com/press/netlify-acquires-gatsby-inc-to-accelerate-the-future-of-composable-web-architectures/
[^3]: Gatsby documentation, hosting/deployment and Gatsby Cloud sunset guidance. https://www.gatsbyjs.com/docs/how-to/previews-deploys-hosting/

## Tags

javascript, react, graphql, static-site-generator, ssg, jamstack, web-framework, content, netlify
