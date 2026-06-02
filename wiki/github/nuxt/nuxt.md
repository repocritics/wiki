# nuxt/nuxt

Nuxt — the full-stack Vue framework. Server-rendering + static-site generation + client-side rendering for Vue apps. Vue's answer to Next.js.

## What it is

A TypeScript meta-framework on top of Vue.js that handles routing, SSR / SSG / CSR / ISR, server-side functions, asset optimization, and i18n out of the box. File-based routing under `pages/`; server endpoints under `server/api/`. Auto-imports for components and composables. Distinguishes itself from raw Vue by being a complete framework with strong conventions.

## Key features

- File-based routing (Vue Router-driven).
- Server-side rendering, static generation, hybrid rendering.
- Server endpoints via `server/api/` for backend logic.
- Nitro engine — deployable to many platforms (Node, Vercel, Netlify, Cloudflare Workers, Deno Deploy).
- Auto-imports for components, composables, utilities.
- Modules ecosystem (`@nuxt/content`, `@nuxt/image`, `@nuxt/ui`, hundreds of community modules).
- TypeScript-first; strict mode supported.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Vue 3 + Vite + Vue Router under the hood.
- Nitro for server build + multi-platform deploy.

## When to reach for it

- You're a Vue team building a full-stack app.
- You want SSR / hybrid rendering for SEO + performance.
- You want strong conventions and minimal boilerplate.

## When *not* to reach for it

- You're React-first — Next.js / Remix are the equivalents there.
- You want full control without framework opinions — bare Vite + Vue + Vue Router is lighter.

## Maturity signal

60k stars, 5.6k forks, MIT, actively maintained under NuxtLabs.

## Alternatives

- Next.js (React).
- Remix (React).
- SvelteKit (Svelte).
- Astro — multi-framework, content-first.

## Tags

typescript, vue, web-framework, frontend, framework, server-side-rendering, static-site-generator, nuxt, mit-license
