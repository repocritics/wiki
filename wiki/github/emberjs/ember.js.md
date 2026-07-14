# emberjs/ember.js

> A batteries-included, convention-over-configuration JavaScript framework for long-lived, ambitious web applications.

[GitHub repo](https://github.com/emberjs/ember.js) ·
[Official website](https://emberjs.com) ·
[License: MIT](https://github.com/emberjs/ember.js/blob/main/LICENSE)

## Overview

Ember.js is an opinionated, full-stack-on-the-client framework that grew out of SproutCore 2.0 and was renamed Ember by Yehuda Katz and Tom Dale in late 2011; the 1.0 release shipped in 2013[^1]. Where React is a view library and Next.js assembles a stack around it, Ember ships the whole stack in one decision: a router, a component/template layer, a reactivity system, a data layer (Ember Data), and a build toolchain (Ember CLI). The trade is explicit — you adopt Ember's conventions wholesale, and in return most of the wiring is done for you and stays stable across years.

The defining cultural feature is **stability without stagnation**, delivered through two mechanisms: a formal RFC process for every user-facing change, and the *edition* system. Editions bundle a set of already-shipped, backward-compatible features into a coherent new programming model with fresh docs. **Octane** (shipped with Ember 3.15, 2019)[^2] replaced the old Ember Object model with native classes, decorators, Glimmer components, angle-bracket invocation (`<MyComponent />`), and **autotracking** reactivity. The next edition, **Polaris**, is in progress and centers on single-file components (`.gjs`/`.gts` with `<template>` tags), first-class TypeScript, and a Vite-based build.

Ember's audience is teams building apps measured in years, not quarters — internal tools, dashboards, SaaS products, and large SPAs where a shared, enforced structure across a rotating roster of engineers matters more than raw flexibility. Its unfashionability is real: the ecosystem is smaller than React's and hiring is harder. The counterweight is that Ember apps written years ago still upgrade forward, and the framework rarely asks you to rewrite.

## Getting Started

```bash
npx ember-cli new my-app
cd my-app
npm start        # dev server (Vite in newer blueprints)
```

```js
// app/components/greeting.gjs — Glimmer single-file component (Polaris style)
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { on } from '@ember/modifier';

export default class Greeting extends Component {
  @tracked count = 0;

  increment = () => this.count++;

  <template>
    <button type="button" {{on "click" this.increment}}>
      Clicked {{this.count}} times
    </button>
  </template>
}
```

Autotracking means `count` re-renders its consumers automatically when mutated — there is no `setState`, no dependency array, and no manual invalidation.

## Architecture / How It Works

Ember is several tightly co-designed subsystems presented as one framework:

- **Router** — the backbone, and Ember's strongest differentiator. Routes are declared in `app/router.js` and map URL segments to route objects that own data loading (`model` hooks), template rendering, and nested UI state. The URL is treated as canonical application state; deep-linking, back/forward, and loading/error substates fall out of the routing model rather than being bolted on.
- **Glimmer** — the rendering engine (a bytecode VM introduced with Ember 2.10, 2016). Templates compile to an opcode program that the VM executes, distinguishing static from dynamic content so only reactive regions update. Templates are Handlebars-derived (`.hbs`) or embedded via `<template>` in `.gjs`/`.gts`.
- **Autotracking** — the reactivity model since Octane. `@tracked` properties form a dependency graph; reading a tracked value during render subscribes that render to it, and mutation marks consumers dirty. This replaced Ember's older observer/computed-property system (`Ember.Object`, `.get`/`.set`).
- **Run loop (Backburner)** — Ember batches DOM updates, data syncing, and timers into scheduled queues so work coalesces within a tick. Mostly invisible now, but it surfaces in tests (`await settled()`) and when integrating non-Ember async.
- **Ember Data** — an optional but default-installed data layer: a normalized identity-mapped store with adapters/serializers abstracting the backend (JSON:API by default). It is powerful and also the most commonly ejected part of the stack.
- **Ember CLI + build** — the classic build was **Broccoli**-based with custom AMD-style module loading. The current direction, **Embroider**, compiles the app into standard ES modules and hands bundling to Vite (or webpack), unlocking tree-shaking, code-splitting, and mainstream tooling.

The subsystems assume each other. This is why Ember feels coherent and why partial adoption is awkward: the router expects Ember components, Ember Data expects the store, and tests expect the run loop.

## Production Notes

**The Broccoli → Embroider migration is the central operational reality.** Legacy apps use the classic build with implicit module resolution; moving to Embroider + Vite is the path to modern bundling, faster builds, and TypeScript ergonomics, but addons relying on classic build hooks or runtime AMD tricks can break. Budget real time for it on any older codebase, and audit addon compatibility first.

**Bundle size.** Ember historically shipped a larger baseline than a hand-assembled React app because the router, run loop, and Glimmer VM come as a set. Embroider's tree-shaking has narrowed this materially, but Ember remains a "framework tax" choice — you pay for structure. It is rarely the right pick for a landing page or a size-critical embed.

**Ember Data is the usual escape hatch.** Teams with GraphQL or bespoke REST shapes frequently drop Ember Data for plain `fetch`, `ember-apollo-client`, or a thin custom store. Nothing in Ember forces Ember Data, but blueprints and much community tutorial content assume it, so going without means more self-navigation.

**Upgrades are smooth by framework standards.** LTS releases and deprecation-with-codemod are the norm; `@tracked` and Octane arrived as additive, opt-in features. The genuinely large jump is *pre-Octane to Octane* (observers/mixins/`Ember.Object` → native classes + autotracking), which is a mental-model change more than a version bump. New code should be Octane/Polaris from day one.

**Testing is a first-class strength.** The `@ember/test-helpers` + QUnit stack gives fast, deterministic application and rendering tests with real routing and a settled-state model, which is one of the most-cited reasons long-lived teams stay.

**Hiring and ecosystem risk.** Fewer job candidates know Ember, and some addons lag the latest edition. Assess the specific addons you depend on against current Ember versions before committing a new project.

## When to Use / When Not

**Use when:**
- You are building a long-lived, routing-heavy SPA (dashboard, admin, SaaS) where URL-as-state and enforced structure pay off over years.
- A rotating team benefits from one prescribed way to do things over per-project architecture debates.
- You value additive, codemod-assisted upgrades over chasing a fast-moving ecosystem.

**Avoid when:**
- You need SSR/RSC-first rendering or edge streaming as a core requirement — that is Next.js/Remix territory, not Ember's.
- Bundle size or a minimal footprint is a hard constraint (marketing sites, embeds, content-first pages).
- Your team's skills, hiring pool, or required libraries are centered on the React ecosystem.

## Alternatives

- angular/angular — the closest peer: batteries-included, opinionated, TypeScript-first. Use Angular when you want a similar all-in-one contract with a larger enterprise ecosystem and stronger SSR support.
- vercel/next.js — use when SSR, React Server Components, and a hosting-integrated stack are the priority rather than a client-owned SPA.
- vuejs/core — use when you want a progressive, less all-encompassing component framework you can adopt incrementally.
- sveltejs/svelte — use when compiled output and minimal runtime size matter more than a prescribed full-stack structure.
- facebook/react — use when you want a view library and intend to assemble routing, data, and build tooling yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| (rename) | 2011-12 | SproutCore 2.0 renamed Ember.js[^1]. |
| 1.0 | 2013-08 | First stable release. |
| 2.0 | 2015-08 | Cleanup release; removed deprecated 1.x APIs. |
| 2.10 | 2016-12 | Glimmer 2 rendering engine. |
| 3.0 | 2018-02 | Dropped long-deprecated APIs; smaller core. |
| 3.15 | 2019-12 | Octane edition: native classes, Glimmer components, autotracking[^2]. |
| 4.0 | 2021-12 | Octane as default; removed pre-Octane legacy paths. |
| 5.0 | 2023 | Raised baselines; continued Embroider direction. |
| 6.0 | 2024-12 | Vite-based builds move toward default; Polaris groundwork. |

## References

[^1]: Ember.js history and origin from SproutCore 2.0. https://en.wikipedia.org/wiki/Ember.js
[^2]: Ember Octane edition announcement. https://blog.emberjs.com/octane-is-here/
[^3]: Ember release model, LTS cadence, and security policy. https://emberjs.com/releases/
[^4]: Embroider — Ember's modern build system. https://github.com/embroider-build/embroider

## Tags

javascript, typescript, framework, spa, frontend, web-framework, mvc, ember, glimmer, convention-over-configuration, routing, reactivity
