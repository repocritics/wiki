# nestjs/nest

NestJS — a progressive Node.js framework for building enterprise-grade server-side apps. Angular-flavored architecture (decorators, DI, modules) on top of Express/Fastify.

## What it is

A TypeScript framework for backend services that brings Angular-style architecture (decorators, dependency injection, modules) to Node.js. Built on top of Express (or Fastify as alternative adapter). Provides primitives for REST, GraphQL, WebSocket, microservices, gRPC, and queue-driven jobs. Heavy adoption in enterprise / Angular-team-flavored backends.

## Key features

- Decorators-driven controllers + services.
- First-class DI with module-based composition.
- Express + Fastify HTTP adapters.
- REST + GraphQL + WebSocket + microservices + gRPC + queue support.
- CLI scaffolding (`nest g controller foo`).
- TypeScript-first; types are not optional.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Express (default) or Fastify HTTP adapter.
- npm package: `@nestjs/*`.

## When to reach for it

- You're an Angular team building Node.js backends and want familiar architecture.
- You want a batteries-included Node framework rather than assembling Express + middleware.
- You're shipping enterprise services where DI + module structure pays off.

## When *not* to reach for it

- You want minimal — Express, Fastify, or Hono are lighter starting points.
- You're allergic to decorators / heavy framework abstractions.

## Maturity signal

76k stars, 8k forks, MIT, actively maintained under NestJS Inc. Open-issues count of 27 is exceptionally low — tight team triage. 8+ years.

## Alternatives

- Express, Fastify, Hono — lighter Node frameworks.
- AdonisJS — Laravel-flavored Node framework.

## Tags

typescript, nodejs, framework, backend, nestjs, mit-license, dependency-injection, microservices
