# angular/angular

Google's TypeScript-first web framework — opinionated, batteries-included, with a strong enterprise foothold.

## What it is

A full-fledged web framework (not just a UI library) — routing, DI, forms, HTTP, state management, and SSR all first-party. TypeScript is the canonical language. The modern era (Angular 14+) introduced standalone components and signals, shedding much of the historical NgModule complexity. Maintained by the Angular team at Google.

## Key features

- Full framework: components, dependency injection, routing, forms, HTTP, animations, change detection.
- Standalone components (no NgModule required) + Signals reactivity (post-Angular 16).
- TypeScript-first; types are not optional.
- Angular CLI for project scaffolding, build, test, lint.
- SSR via Angular Universal; SSG via prerendering.
- Material Components, CDK, Angular DevTools as first-party.
- MIT-licensed.

## Tech stack

- TypeScript primary; Angular itself is written in TypeScript.
- RxJS deeply integrated (Observables in HTTP, forms, router).
- Distributed as npm `@angular/*` packages.

## When to reach for it

- You're building enterprise-scale apps where DI, RxJS, and strong opinions about structure are an asset.
- You're staffing from an enterprise-Java background — Angular's class-based, DI-driven model maps naturally.
- You want a batteries-included framework rather than assembling pieces.

## When *not* to reach for it

- You want a thin library — React, Vue, Svelte are lighter starting points.
- You want minimal RxJS — Angular's HTTP and reactive primitives lean heavily on Observables; signals are reducing this but RxJS remains.
- You want max ecosystem velocity — Angular's user community is smaller than React's.

## Maturity signal

100k stars, 27k forks, MIT, actively maintained. 12+ years under Google. Open-issues count of 1,175 is moderate. The standalone-components + signals migration is the most-significant architectural shift in years, gradually closing the developer-ergonomics gap with React/Vue.

## Alternatives

- React, Vue, Svelte, Solid — use for less opinionated alternatives.
- AnalogJS — use for an Angular meta-framework akin to Next.js.

## Tags

angular, typescript, web-framework, frontend, javascript, framework, mit-license
