# adonisjs/core

> A TypeScript-first, batteries-included MVC framework for Node.js — the closest thing the Node ecosystem has to Laravel.

[GitHub repo](https://github.com/adonisjs/core) ·
[Official website](https://adonisjs.com) ·
[License: MIT](https://github.com/adonisjs/core/blob/7.x/LICENSE.md)

## Overview

AdonisJS is a full-stack MVC web framework for Node.js, created by Harminder Virk (thetutlage) and first published in 2015[^1]. Where most of the Node backend world is assembled from micro-libraries (Express + an ORM + a validator + a test runner + a DI container), AdonisJS ships all of those as tightly integrated first-party packages under a single opinionated structure. Its explicit design reference is Laravel: an IoC container, service providers, an Active Record ORM, a template engine, a first-class CLI, and convention-over-configuration project layout[^2].

The framework is TypeScript-first, not TypeScript-compatible: controllers, models, validators, and the container are all designed to give end-to-end type inference without hand-written type annotations at every boundary. This is the source of both its appeal and its friction. The integration is deep — Auth, Bouncer (authorization), Lucid (ORM), VineJS (validation), Edge (templates), and Ace (CLI) are meant to be used together and share conventions — which is productive if you adopt the whole stack and awkward if you want to swap one piece for a popular external library.

The defining tension is **integration vs. ecosystem gravity**. AdonisJS gives a coherent, well-typed, batteries-included experience, but it swims against Node's culture of small composable packages. Its community, plugin selection, and hiring pool are a fraction of Express, Nest, or Fastify, and each major version has been a substantial rewrite that invalidated older tutorials and StackOverflow answers.

## Getting Started

```bash
npm init adonisjs@latest my-app
# prompts: starter kit (api / web / slim), auth guard, database driver
cd my-app
node ace serve --watch
```

```ts
// start/routes.ts — lazy controller import is the v6 convention
import router from '@adonisjs/core/services/router'

const PostsController = () => import('#controllers/posts_controller')

router.get('/posts', [PostsController, 'index'])
```

```ts
// app/controllers/posts_controller.ts
import type { HttpContext } from '@adonisjs/core/http'

export default class PostsController {
  async index({ response }: HttpContext) {
    return response.ok({ posts: [] })
  }
}
```

## Architecture / How It Works

The heart of the framework is an **IoC container** and a **service provider** lifecycle. Providers register bindings during boot; application code resolves them either explicitly or through constructor injection, and the container carries the type information so resolved services are fully typed[^2]. The HTTP layer is AdonisJS's own server (not Express under the hood), driving a middleware pipeline into controllers via `HttpContext`.

Project structure is by convention: `start/` for boot-time files (routes, kernel, middleware registration), `app/` for controllers, models, middleware and services, `config/` for typed config modules, `database/` for migrations and seeders. The **Ace CLI** (`node ace`) scaffolds and runs everything — code generators, migrations, REPL, custom commands.

Notable first-party pieces, each a separate package coordinated by `core`:

- **Lucid** — Active Record ORM built on Knex, with models, relationships, migrations, and a query builder. Migrations are explicit files, not schema-diffing.
- **VineJS** — the validation library introduced in v6, replacing the older bundled validator. Schemas infer TypeScript types.
- **Edge** — the server-side template engine for the `web` starter kit.
- **Auth + Bouncer** — session, basic-auth, and access-token guards plus policy-based authorization.
- **Japa** — the test runner AdonisJS ships and integrates, rather than Jest or Vitest.

Version 6 (2024) moved the framework to **ESM-only** and adopted Node.js **subpath import maps** — the `#controllers/*`, `#models/*` style specifiers you see above are declared in `package.json`'s `imports` field, not TypeScript path aliases[^3]. This is a real architectural choice, not sugar: it affects how bundlers, test runners, and IDEs resolve modules, and it is one of the first things that surprises newcomers.

## Production Notes

**Each major version is a migration project.** v4 (JS-era), v5 (the TypeScript rewrite, 2021), and v6 (ESM + restructured core, 2024) were not incremental. Folder layout, import style, and bundled packages changed across them, so upgrading is a deliberate porting effort and older online answers are frequently wrong for the version you are running[^3]. Pin your docs to your exact major version.

**ESM-only in v6+.** There is no CommonJS build. Dependencies that are CJS-only, tooling that assumes `require`, and some serverless bundlers need explicit handling. Run on a modern Node LTS.

**Ecosystem size is the honest constraint.** With ~19k stars the project is healthy and actively maintained (the `7.x` line is the current default branch), but the third-party plugin catalog, community Q&A volume, and available hires are far smaller than Express/Nest/Fastify. For anything off the golden path you will read source code rather than blog posts.

**Lucid is Knex-based Active Record.** Mature and pleasant, but it is not Prisma: no schema-diff migrations, no generated client, and a different mental model. Teams expecting Prisma-style DX should evaluate Lucid directly before committing, or wire Prisma/Drizzle in manually and give up some of the integrated typing.

**Opinion is the value; fighting it is the cost.** The payoff comes from adopting the whole stack — container, Lucid, Vine, Auth, Ace. If your instinct is to replace half the first-party packages with the Node micro-library you already know, you lose most of what distinguishes AdonisJS and inherit the integration work anyway.

## When to Use / When Not

**Use when:**
- You want a coherent, well-typed, Laravel-style backend and are happy to adopt the framework's conventions wholesale.
- You value integrated Auth, ORM, validation, CLI, and testing over à la carte assembly.
- Your team prizes structure and end-to-end type inference over ecosystem breadth.

**Avoid when:**
- You need the largest possible plugin ecosystem and hiring pool — Express or Nest win there.
- You want to compose your own stack from independent micro-libraries; AdonisJS's integration is wasted effort then.
- You need long-term-stable conventions with rare breaking changes — the major-version rewrites are real.
- You require CommonJS or a bundler/runtime that struggles with ESM-only, subpath-import packages.

## Alternatives

- nestjs/nest — the larger TypeScript backend framework; decorator/DI-heavy and Angular-inspired, but you assemble ORM, validation, and testing yourself. Use when you want a big ecosystem and modular architecture.
- expressjs/express — minimal and unopinionated with the deepest ecosystem. Use when you want to own every architectural decision.
- fastify/fastify — schema-first and performance-focused with a plugin model. Use when throughput and JSON-schema validation are priorities.
- balderdashy/sails — the older Node MVC/Rails-inspired framework. Use when you want convention-driven MVC but on a more traditional (CJS) footing.
- adonisjs/lucid — AdonisJS's own ORM, usable to understand the data layer independently; pair it with Prisma or Drizzle as external alternatives when Active Record is not the right fit.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2015 | Initial release as a Node.js MVC framework[^1]. |
| 4.x | 2017–2019 | "AdonisJs" era, largely JavaScript, IoC container + Lucid established. |
| 5.0 | 2021 | Full TypeScript rewrite; typed container and config[^2]. |
| 6.0 | 2024 | ESM-only, restructured `core`, subpath import maps, VineJS validation[^3]. |
| 7.x | in development | Current default branch as of 2026. |

## References

[^1]: `adonisjs/core` repository, created 2015-08-15. https://github.com/adonisjs/core
[^2]: AdonisJS documentation — Introduction and IoC container. https://docs.adonisjs.com/guides/getting-started/introduction
[^3]: AdonisJS documentation — folder structure, ESM, and subpath imports (v6). https://docs.adonisjs.com/guides/getting-started/folder-structure

## Tags

typescript, nodejs, web-framework, mvc, backend, api, orm, full-stack, ioc-container, laravel-inspired, esm
