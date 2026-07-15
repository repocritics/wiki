# aurelia/aurelia

> Aurelia 2 — a standards-aligned, convention-driven front-end framework that stays out of your code, with a small but committed following.

[GitHub repo](https://github.com/aurelia/aurelia) ·
[Official website](https://aurelia.io) ·
[License: MIT](https://github.com/aurelia/aurelia/blob/master/LICENSE)

## Overview

Aurelia is a component-based JavaScript/TypeScript front-end framework. This repository is the **Aurelia 2** monorepo: a ground-up TypeScript rewrite of the original Aurelia 1 (which lived at `aurelia/framework`)[^1]. Its goals carry over from v1: align with web platform standards, prefer convention over configuration, and keep the framework out of the way of plain classes. A component is a vanilla class plus an HTML template — no mandatory `render()`, no JSX, no framework-specific base class. Aurelia was originally created by Rob Eisenberg (previously the author of Durandal and Caliburn.Micro), and that lineage shows in its MVVM leanings and its focus on data binding as the central abstraction.

The defining tension is not technical — it is adoption. Aurelia is well-regarded for its clean dependency injection, expressive binding language, and genuine two-way binding, but it has a fraction of the mindshare of React, Vue, or Angular. The star count on this monorepo (~1.5k) reflects that: an order of magnitude smaller than the frameworks it competes with. Aurelia 2 has also been developed under a **beta** label for an extended period — the README itself warns that public-API surface is still subject to breaking changes[^2]. Evaluating it means trading a smaller ecosystem and slower release cadence for a codebase that is pleasant to work in and closely tracks native browser APIs.

## Getting Started

The maintainers' preferred scaffolder is the `makes` generator:

```bash
npx makes aurelia
```

This prompts for TypeScript/JavaScript, a bundler (Webpack/Vite), and testing setup, then produces a runnable project. A minimal component is a class and a template of the same name:

```ts
// my-app.ts
export class MyApp {
  message = "Hello world";
  count = 0;

  increment() {
    this.count++;
  }
}
```

```html
<!-- my-app.html -->
<h1>${message}</h1>
<button click.trigger="increment()">Clicked ${count} times</button>
<input value.bind="message">
```

The binding syntax is the core of the framework: `${...}` for interpolation, `.bind` for property binding (two-way by default on form controls), `.trigger`/`.delegate` for events, `repeat.for` for iteration, and `if.bind` for conditional rendering. Binding behaviors chain with `&` (e.g. `value.bind="q & debounce:500"`).

## Architecture / How It Works

Aurelia 2 is split into small, independently published packages under the `@aurelia/*` scope rather than shipped as one bundle:

- **`@aurelia/kernel`** — the dependency injection container, service registry, and platform abstractions. DI is a first-class citizen: dependencies resolve by constructor injection using decorators or a `static inject` array. This container is usable on its own, independent of the rendering layer.
- **`@aurelia/runtime`** — the observation and binding engine, agnostic of the DOM.
- **`@aurelia/runtime-html`** — the DOM renderer, custom element and custom attribute definitions, and the template compiler.
- **`@aurelia/router` / `@aurelia/router-lite`** — routing. Aurelia 2 shipped two routers with different models (a component-oriented "direct" router and a more conventional configured router); choosing between them is an early architectural decision.
- **`@aurelia/metadata`, `@aurelia/platform`, `@aurelia/expression-parser`** and others provide the supporting layers.

**Observation** is the interesting internal. Rather than dirty-checking the whole view tree, Aurelia installs per-property observers and only re-evaluates bindings whose dependencies changed — using getter/setter interception for plain objects, dedicated array/set/map observers for collections, and an optional Proxy-based mode. The template compiler turns HTML into an instruction set once, so repeated renders (e.g. inside `repeat.for`) reuse compiled instructions.

Convention over configuration is enforced by build-time plugins (Webpack, Vite): a class paired with a same-named `.html` file is wired into a custom element automatically, without a decorator. Dropping the convention plugin falls back to explicit `@customElement` registration — the "pure JIT" setups in the repo's `examples` folder show this path.

## Production Notes

**Beta status is the headline caveat.** Aurelia 2's public API has been marked beta for a long stretch, and the README explicitly notes untested API surface and further breaking changes[^2]. This is manageable for greenfield apps that can absorb churn, but it is the single biggest reason to hesitate on a long-lived commercial project.

**Ecosystem depth.** Because the community is small, you will find fewer ready-made component libraries, Stack Overflow answers, and third-party integrations than for React/Vue/Angular. When something breaks, expect to read source or ask in the project's Discord/Discourse rather than find an existing answer. Hiring developers already fluent in Aurelia is also harder.

**Convention plugin coupling.** The zero-boilerplate experience depends on the Webpack/Vite convention plugin. If you adopt an unusual bundler or a non-standard file layout, you lose the automatic view/view-model pairing and must register components explicitly — a footgun for teams that customize their build early.

**Two-way binding defaults.** `value.bind` is two-way on form elements by default. This is convenient but means unintended mutation of view-model state is easy; teams coming from one-way-data-flow frameworks should be deliberate about `.to-view` / `.one-time` where mutation is not wanted.

**Migration from Aurelia 1.** Aurelia 2 is a rewrite, not a drop-in upgrade. Concepts transfer, but the package layout, lifecycle hooks, and router APIs differ enough that a v1→v2 migration is a porting exercise, not a version bump.

## When to Use / When Not

**Use when:**
- You value plain-class components and minimal framework intrusion over ecosystem size.
- You want strong, standards-aligned two-way data binding and first-class DI without assembling them yourself.
- Your team can tolerate beta churn and self-supporting via source/community.
- You are building an internal or long-horizon app where developer ergonomics matter more than off-the-shelf component availability.

**Avoid when:**
- You need a large hiring pool, a deep third-party component ecosystem, or abundant tutorials — React/Vue/Angular win decisively here.
- API stability guarantees are a hard requirement today (beta status).
- You want React Server Components / SSR-first architectures as a supported happy path; that is not Aurelia's focus.

## Alternatives

- vuejs/core — comparable template-plus-class-instance model and reactivity, far larger ecosystem; use Vue when community size and stability outweigh Aurelia's ergonomics.
- angular/angular — pick Angular when you want an all-in-one, opinionated, enterprise-supported framework with built-in DI and are fine with its size.
- sveltejs/svelte — use Svelte when you want compile-time reactivity and minimal runtime rather than Aurelia's observer-based runtime.
- vercel/next.js — use Next when React and a first-class SSR/hosting story are requirements Aurelia does not target.
- aurelia/framework — the original Aurelia 1; use it only for maintaining existing v1 apps, not new work.

## History

| Version | Date | Notes |
|---------|------|-------|
| Aurelia 1.0 | 2018-07 | First stable release of the original framework (`aurelia/framework`)[^1]. |
| Aurelia 2 repo created | 2018-02 | This monorepo begins as the v2 rewrite[^3]. |
| Aurelia 2 beta | 2021 onward | Public beta packages under `@aurelia/*`, ongoing breaking changes[^2]. |

## References

[^1]: Aurelia blog, "Aurelia 1.0" release announcement (2018). https://aurelia.io/blog
[^2]: aurelia/aurelia README — "Please keep in mind that Aurelia 2 is still in beta." https://github.com/aurelia/aurelia
[^3]: GitHub API repository metadata for aurelia/aurelia, `created_at` 2018-02-15. https://github.com/aurelia/aurelia

## Tags

typescript, javascript, front-end-framework, spa, data-binding, dependency-injection, mvvm, web-components, convention-over-configuration, aurelia
