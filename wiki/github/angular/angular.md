# angular/angular

> Google's TypeScript-first web application framework — batteries-included, opinionated, and enterprise-oriented.

[GitHub repo](https://github.com/angular/angular) ·
[Official website](https://angular.dev) ·
[License: MIT](https://github.com/angular/angular/blob/main/LICENSE)

## Overview

Angular is a component-based front-end framework built and maintained by Google[^1]. It is a full framework rather than a view library: routing, forms, HTTP, dependency injection, a build system, and testing scaffolding all ship as first-party packages under one versioned umbrella. This is the defining tradeoff — you adopt an integrated, opinionated stack with fewer decisions to make and fewer third-party glue libraries, at the cost of a larger baseline and a stronger set of conventions to learn.

The name is a persistent source of confusion. **AngularJS** (the 1.x line, 2010) and **Angular** (2.0 and later, 2016) are different frameworks. Angular 2 was a full rewrite in TypeScript with an incompatible component model; there is no in-place upgrade path, only migration[^2]. AngularJS reached end-of-life in January 2022 and is unrelated to anything shipped from this repository. When people say "Angular is heavy/complex" they sometimes mean one and sometimes the other; treat pre-2016 opinions with suspicion.

Angular is most common in large, long-lived enterprise applications — internal line-of-business apps, dashboards, and regulated-industry front-ends — where TypeScript-by-default, dependency injection, and a stable upgrade story matter more than minimal bundle size. Since roughly 2023 the framework has been in a sustained reactivity overhaul: **signals**, standalone components, a new control-flow template syntax, and optional zoneless change detection are progressively replacing the Zone.js-and-NgModules architecture that defined Angular 2 through 15[^3].

## Getting Started

```bash
npm install -g @angular/cli
ng new my-app        # prompts: routing? stylesheet format? SSR/SSG?
cd my-app
ng serve             # dev server with HMR
```

A standalone component using signals (modern idiom, no NgModule):

```ts
// src/app/counter.component.ts
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <button (click)="increment()">Count: {{ count() }}</button>
    <p>Doubled: {{ doubled() }}</p>
  `,
})
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);

  increment() {
    this.count.update(n => n + 1);
  }
}
```

Bootstrapping without a root module:

```ts
// src/main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { CounterComponent } from './app/counter.component';

bootstrapApplication(CounterComponent);
```

## Architecture / How It Works

**Components, templates, DI.** The unit of composition is the component: a TypeScript class annotated with `@Component`, a template (inline or `.html`), and styles. Angular's hierarchical dependency injection resolves services by walking an injector tree, which is central to how testing, lazy loading, and platform abstraction work. This DI system is more elaborate than in most front-end frameworks and is one reason Angular feels closer to a server framework like Spring than to React.

**The compiler (Ivy).** Templates are not interpreted at runtime; the Angular compiler transforms them into imperative rendering instructions. **Ivy**, the rewrite of the compiler and runtime that became the default in Angular 9 (2020), enabled better tree-shaking, faster builds, and smaller output than the previous View Engine[^4]. Ahead-of-time (AOT) compilation is the production default.

**Change detection — the pivot point.** Historically Angular detected changes via **Zone.js**, which monkey-patches browser async APIs (`setTimeout`, `addEventListener`, `Promise`, XHR) so the framework knows when to re-check the component tree. This "it just works" model costs a full-tree check on every async event and is the source of much of Angular's runtime overhead and debugging difficulty. The signals initiative reverses this: **signals** are fine-grained reactive primitives that let Angular update only the views that actually depend on changed state, and enable **zoneless** change detection where Zone.js is dropped entirely[^3]. As of the 2024–2026 releases this migration is real but partial — signals are stable, zoneless is stabilizing, and RxJS-plus-Zone.js code still runs unchanged.

**RxJS.** Reactive Extensions for JavaScript is a first-party dependency used for HTTP, router events, and reactive forms. It is a well-known source of Angular's learning curve; the signals work is partly an effort to make RxJS optional for common cases rather than load-bearing.

**Build system.** The CLI's builder moved from webpack to an **esbuild**-based pipeline with a Vite-backed dev server (the `application` builder), which substantially cut cold build and rebuild times versus the older webpack setup. Server-side rendering and hydration are handled by `@angular/ssr`, with non-destructive hydration and, in recent versions, incremental hydration for deferred content.

**Template evolution.** Angular 17 introduced a built-in control-flow syntax (`@if`, `@for`, `@switch`) and deferrable views (`@defer`) that replace the older structural directives (`*ngIf`, `*ngFor`) with a form the compiler handles directly and that is easier to lazy-load.

## Production Notes

**NgModules → standalone.** For most of its life Angular organized code into `@NgModule` declarations. Standalone components (stable in v15, the default in new projects from v19) remove that layer. Mature codebases carry both; the `ng generate @angular/core:standalone` schematic automates most of the migration but mixed-mode apps are common and add cognitive load.

**Upgrade cadence.** Angular ships two major versions per year with a documented deprecation-and-support policy[^6]. The upgrade treadmill is real, but `ng update` runs schematic migrations that mechanically rewrite deprecated APIs, which makes major upgrades far less painful than the version number implies — provided you stay reasonably current. Teams that skip several majors lose the migration path and face manual work.

**Bundle size and initial load.** Angular's baseline is larger than React or Svelte for a trivial app. This matters for public, latency-sensitive, first-load-critical sites and matters much less for authenticated enterprise apps where the framework cost is amortized. Mitigations: route-level lazy loading, `@defer` blocks, and SSR with hydration.

**Zone.js debugging.** Under the classic model, stack traces are polluted by Zone.js frames and `ExpressionChangedAfterItHasBeenCheckedError` is a notorious dev-mode error rooted in the change-detection cycle. Moving to signals and zoneless removes much of this class of problem but requires code written against the new primitives.

**RxJS as an onboarding cost.** Reactive forms and HTTP idioms assume Observable fluency. Subscription leaks (forgetting to unsubscribe) are a recurring production bug; `takeUntilDestroyed`, the `async` pipe, and increasingly signals are the standard defenses.

**Version numbering.** Angular jumped from 2 to 4, skipping 3, to align the core version with the router package (already at v3) — not a rewrite. Every major since has been an incremental, migratable release, not a v1→v2-style break.

## When to Use / When Not

**Use when:**
- You're building a large, long-lived application and want an integrated, first-party stack (router, forms, HTTP, DI, i18n, testing) with one version to track.
- The team values TypeScript-by-default, strong conventions, and a documented upgrade policy over minimal footprint.
- You want dependency injection and a structured architecture that scales across many contributors.
- You need enterprise support signals: predictable release cadence, long-term-support windows, and a large hiring pool.

**Avoid when:**
- You're shipping a small, static, or content-first site where framework weight dominates — Astro or Svelte are leaner.
- You want maximum ecosystem flexibility and minimal opinions — React lets you assemble your own stack.
- Your team is small and wants fast throwaway-code velocity; Angular's conventions and RxJS/DI concepts are overhead at that scale.
- You're maintaining an AngularJS (1.x) app and hoping for an in-place upgrade — there isn't one; it's a rewrite/migration either way.

## Alternatives

- facebook/react — a view library, not a framework; more flexible, smaller core, but you assemble routing/data/build yourself.
- vuejs/core — gentler learning curve and lighter baseline; a framework-ish middle ground between React and Angular.
- sveltejs/svelte — compiler-first with minimal runtime; far smaller output, less structure for very large teams.
- vercel/next.js — if you want SSR/RSC and a hosting-integrated meta-framework on the React side.
- emberjs/ember — the other opinionated, convention-heavy, batteries-included framework; similar philosophy, smaller ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| AngularJS 1.0 | 2012-06 | The original framework. Separate, now EOL (Jan 2022)[^2]. |
| 2.0 | 2016-09 | Full TypeScript rewrite; incompatible component model[^1]. |
| 4.0 | 2017-03 | Skipped v3 to align with the router package version. |
| 6.0 | 2018-05 | `ng update` / `ng add`, unified version scheme. |
| 9.0 | 2020-02 | Ivy compiler/renderer becomes the default[^4]. |
| 14.0 | 2022-06 | Standalone components (preview), typed reactive forms. |
| 15.0 | 2022-11 | Standalone components stable. |
| 16.0 | 2023-05 | Signals (developer preview), SSR hydration, esbuild dev preview[^3]. |
| 17.0 | 2023-11 | Built-in control flow (`@if`/`@for`), `@defer`, new `angular.dev` docs, esbuild/Vite build[^5]. |
| 18.0 | 2024-05 | Zoneless change detection (experimental), event replay. |
| 19.0 | 2024-11 | Standalone the default, incremental hydration, signal API refinements. |

Major releases continue on a roughly six-month cadence[^6].

## References

[^1]: Angular team blog. https://blog.angular.dev/
[^2]: Angular team, "Discontinued Long Term Support for AngularJS". https://blog.angular.dev/discontinued-long-term-support-for-angularjs-cc066b82e2b1
[^3]: Angular, "Angular Signals" guide. https://angular.dev/guide/signals
[^4]: Angular blog, "Version 9 of Angular Now Available — Project Ivy has arrived". https://blog.angular.dev/version-9-of-angular-now-available-project-ivy-has-arrived-23c97b63cfa3
[^5]: Angular blog, "Introducing Angular v17". https://blog.angular.dev/introducing-angular-v17-4d7033312e4b
[^6]: Angular versioning and releases. https://angular.dev/reference/releases

## Tags

typescript, web-framework, frontend, spa, google, dependency-injection, signals, rxjs, component-based, javascript, angular
