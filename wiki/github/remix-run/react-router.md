# remix-run/react-router

React Router — declarative routing for React. The default routing library for React apps; merged with Remix to form a unified routing + framework story.

## What it is

A TypeScript routing library for React: route definitions, nested routes, dynamic parameters, navigation hooks. Maintained by Remix Software (now part of Shopify). React Router v7 unified the React Router library + Remix framework into a single package — projects can use it as a library (`react-router`) or as a framework (`react-router-dom` + the framework conventions).

## Key features

- Nested route definitions with shared layouts.
- Loaders + actions for data fetching at the route level (framework mode).
- Type-safe route params.
- Library mode (just routing) or framework mode (data + SSR + nested layouts).
- TypeScript-first.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- npm package: `react-router-dom` (or `react-router` for non-web targets).

## When to reach for it

- You're building any React app that needs client-side routing.
- You want to upgrade from Remix → React Router v7's framework mode.

## When *not* to reach for it

- You're on Next.js — Next's built-in routing is the canonical choice.
- You want a tiny router — `wouter` or `@tanstack/react-router` may fit.

## Maturity signal

56k stars, 11k forks, MIT, actively maintained under Remix / Shopify.

## Alternatives

- Next.js routing — for Next-based apps.
- TanStack Router — newer TS-first alternative.
- wouter — minimal hooks-based router.

## Tags

react, typescript, routing, library, mit-license, react-router, framework
