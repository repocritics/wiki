# remix-run/remix

> The repo that shipped the Remix v1–v2 React framework (since merged into React Router 7) — and now hosts Remix 3, a React-free, web-standards full-stack toolkit currently in beta.

[GitHub repo](https://github.com/remix-run/remix) ·
[Official website](https://remix.run) ·
[License: MIT](https://github.com/remix-run/remix/blob/main/LICENSE)

## Overview

Remix began as a full-stack React framework by Michael Jackson and Ryan Florence, the authors of React Router. It launched in 2020 under a paid license and was open-sourced under MIT with v1.0 in late 2021[^1]. Its thesis — routes as the unit of data loading (`loader`/`action`), progressive enhancement via plain HTML forms, and Web Fetch API standards instead of Node-specific abstractions — pushed the whole React ecosystem toward server-first patterns. Shopify acquired the team in October 2022[^2].

The project has since been reinvented twice, and the repo reflects that. In May 2024 the team announced that Remix v2's framework features were merging back into React Router[^3]; React Router v7 (November 2024) is the supported continuation of Remix v2 apps, and framework development moved to the react-router repo[^4]. Then in 2025 the "Wake up, Remix" post announced Remix 3: not a v2 successor but a new project under the same name — a monorepo of ~45 standalone TypeScript packages built strictly on Web APIs, with its own view layer (reconciler and component model) and no React dependency[^5].

Read the numbers accordingly: the ~33k stars were mostly earned by the v1/v2 framework, while the low open-issue count (62) reflects the repo's reset for Remix 3, not mature stability. The repo is very actively developed (pushed within the last day at time of writing), but what is being developed is a beta.

## Getting Started

Remix 3 is distributed under the `next` dist-tag; there is no stable release yet.

```bash
npm install remix@next          # umbrella package
npx remix@next new my-remix-app # scaffold an app with the CLI
```

Every package is also usable standalone. Example with the Fetch-API server adapter for Node.js:

```ts
import * as http from "node:http";
import { createRequestListener } from "@remix-run/node-fetch-server";

const server = http.createServer(
  createRequestListener((request) => new Response("Hello, world!"))
);
server.listen(3000);
```

For a production app today, the practical path is React Router v7 in framework mode, not this repo.

## Architecture / How It Works

Remix 3 is organized around six stated principles[^5], three of which have real architectural teeth:

- **"Religiously runtime."** No bundler, no compiler, no typegen. All packages must work without static analysis, and tests run unbundled. This is a direct repudiation of the Vite-coupled Remix v2 architecture and of the wider RSC/compiler direction Next.js has taken.
- **Web APIs as the substrate.** Web Streams instead of `node:stream`, `Uint8Array` instead of `Buffer`, Web Crypto instead of `node:crypto`, `File`/`Blob` instead of runtime-specific file types. The result is portability across Node.js, Bun, Deno, and Cloudflare Workers without adapter layers.
- **Zero-dependency ambition.** Dependencies are wrapped completely with the stated goal of eventually replacing them with in-house packages.

The monorepo contains the building blocks a framework normally hides: `fetch-router` (Fetch-API router), `route-pattern` (typed URL matching), `multipart-parser` and `form-data-parser` (streaming uploads), `session` / `cookie` / `headers`, a middleware suite (CORS, CSRF, compression, auth, static files), `file-storage` with an S3 backend, `data-table` (typed relational query toolkit with SQLite/Postgres/MySQL adapters), a test framework, and `ui` — a from-scratch view layer with its own reconciler. The `remix` package composes them for distribution.

Two consequences are worth underlining. First, Remix 3 is not a React framework; `ui` replaces React entirely, which means the React component ecosystem does not carry over. Second, the repo is explicitly "model-first": source, docs, and abstractions are optimized for LLM consumption, and the repo ships agent skills that the CLI copies into generated apps. Whether that principle produces better human-facing APIs is an open question.

## Production Notes

- **Do not confuse the three Remixes.** "Remix" currently names (a) legacy v1/v2 apps, (b) React Router v7 framework mode (the supported successor), and (c) the Remix 3 beta in this repo. Tutorials, Stack Overflow answers, and LLM training data mix them freely; check dates and imports before trusting any snippet.
- **Remix v2 is maintenance-only.** The official migration path is React Router v7, designed as a non-breaking upgrade for v2 apps that had enabled all future flags[^3]. Staying on v2 long-term means staying on an end-of-lifed line.
- **Remix 3 is beta software.** APIs under the `next` tag can and do change between releases. The `preview/main` git installs advertised in the README are bleeding-edge by definition. No stable 3.0 release date is committed.
- **The view layer is unproven.** A new reconciler and component model has none of the ecosystem, devtools, or battle-testing of React, Vue, or Svelte. Betting a product on it today is a research decision, not an engineering one.
- **Standalone packages are the lower-risk entry point.** Pieces like `multipart-parser`, `cookie`, `headers`, and `node-fetch-server` are small, web-standard, and useful in any Fetch-API server regardless of whether Remix 3 the framework succeeds.
- **Governance concentration.** Direction is set by a small Shopify-funded team that has now changed course twice in four years (framework → merged into React Router → new framework). That agility cut both ways for v2 adopters, who absorbed a rebrand-scale migration mid-lifecycle.

## When to Use / When Not

**Use when:**
- You want to evaluate a web-standards, no-bundler full-stack direction early and can tolerate beta churn.
- You need one of the standalone packages (streaming multipart parsing, typed route patterns, Fetch-API middleware) in a Node/Bun/Deno/Workers server.
- You are maintaining an existing Remix v1/v2 app and need the historical source.

**Avoid when:**
- You are starting a production React app: use React Router v7 framework mode or Next.js instead — this repo's framework is not stable.
- You need a large component ecosystem; Remix 3's `ui` layer has none yet.
- You need long-horizon API stability; the project's own history argues against assuming it.

## Alternatives

- remix-run/react-router — the direct successor for Remix v1/v2 apps; use it for any production Remix-style React app today.
- vercel/next.js — the dominant React framework; use it when you want React Server Components and the largest ecosystem.
- honojs/hono — mature web-standards Fetch-API server framework; use it when you want Remix 3's runtime portability story without the beta framework.
- TanStack/router — typesafe React routing with TanStack Start for full-stack; use it when type-safety of routes is the priority.
- sveltejs/kit — full-stack framework outside React; use it when you want a stable non-React framework rather than a beta one.

## History

| Version | Date | Notes |
|---------|------|-------|
| Preview | 2020 | Launched as a paid "supporter preview" license. |
| 1.0 | 2021-11 | Open-sourced under MIT[^1]. |
| — | 2022-10 | Shopify acquires the Remix team[^2]. |
| 2.0 | 2023-09 | v2 ships; future-flags upgrade model, Vite support followed. |
| — | 2024-05 | Merge into React Router announced[^3]. |
| RR 7.0 | 2024-11 | React Router v7 ships Remix's framework features; framework work moves to the react-router repo[^4]. |
| 3.0 beta | 2025 | "Wake up, Remix": repo reset as a React-free, web-standards package toolkit[^5]. |

## References

[^1]: Remix blog, "Remix v1". https://remix.run/blog/remix-v1
[^2]: Remix blog, "Remixing Shopify". https://remix.run/blog/remixing-shopify
[^3]: Remix blog, "Merging Remix and React Router" — 2024-05. https://remix.run/blog/merging-remix-and-react-router
[^4]: Remix blog, "React Router v7". https://remix.run/blog/react-router-v7
[^5]: Remix blog, "Wake up, Remix!". https://remix.run/blog/wake-up-remix

## Tags

typescript, javascript, web-framework, full-stack, web-standards, fetch-api, server-side-rendering, monorepo, beta, react-router
