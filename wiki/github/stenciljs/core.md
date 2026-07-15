# stenciljs/core

> A compiler that turns TypeScript + JSX classes into standards-based Web Components, so one component set can ship to any framework or none.

[GitHub repo](https://github.com/stenciljs/core) ·
[Official website](https://stenciljs.com) ·
[License: MIT](https://github.com/stenciljs/core/blob/main/LICENSE.md)

## Overview

Stencil is a build-time compiler, not a runtime framework. You author components as TypeScript classes with decorators (`@Component`, `@Prop`, `@State`) and a JSX `render()` method; Stencil compiles them ahead-of-time into custom elements that conform to the Web Components standard[^1]. The output is framework-agnostic: the same component runs unwrapped in a plain HTML page, or through generated wrappers in React, Angular, and Vue. It is built and maintained by the Ionic team, and Ionic Framework's own component library is built on Stencil — that dogfooding is the main reason the project is stable and long-lived.

The defining tradeoff is **compiler over runtime**. Because component logic is resolved at build time, the shipped runtime is small (a compact virtual-DOM renderer plus a lazy loader) and there is no large framework payload. The cost is that Stencil owns your build: it wraps the TypeScript compiler API and Rollup, ships its own Jest-based test runner, and expects you to configure "output targets" rather than a bundler directly. You trade bundler control for a curated pipeline. Teams that want a design system consumed by multiple app frameworks are the sweet spot; teams that want to drop components into an existing Vite/webpack build with full control feel the friction.

The repository moved from `ionic-team/stencil` to `stenciljs/core` during an org restructure; the package on npm remains `@stencil/core` and imports are unchanged[^2]. As of mid-2026 the project has ~13k stars and is actively maintained on a steady point-release cadence (latest 4.x line), though development velocity is deliberate rather than fast-moving — it is mature infrastructure, not a churning framework.

## Getting Started

```bash
npm init stencil
# prompts: component / app / ionic-pwa starter
```

A component is a decorated class with a JSX render:

```tsx
import { Component, Prop, h } from '@stencil/core';

@Component({
  tag: 'my-greeting',
  styleUrl: 'my-greeting.css',
  shadow: true,            // render into Shadow DOM (style isolation)
})
export class MyGreeting {
  @Prop() first: string;
  @Prop() last: string;

  render() {
    return <div>Hello, {this.first} {this.last}</div>;
  }
}
```

Used like any HTML element, in any (or no) framework:

```html
<my-greeting first="Stencil" last="JS"></my-greeting>
```

## Architecture / How It Works

Stencil is an ahead-of-time compiler layered on top of the TypeScript compiler and Rollup. The decorators are compile-time metadata, not runtime reflection — the compiler reads them to generate a component descriptor and then strips them, so there is no `reflect-metadata` dependency in shipped code. `render()` returns hyperscript (`h(...)`), which a small bundled virtual-DOM runtime diffs and applies.

The most important concept is **output targets**, configured in `stencil.config.ts`. The same source compiles to different distributables:

- **`dist` (lazy)** — the classic default. Ships a tiny loader plus one lazily-loaded chunk per component. Components register themselves and are fetched on first use. Smallest initial payload for a large library; adds a loader/proxy indirection layer.
- **`dist-custom-elements`** — modern, tree-shakable standard custom elements with no lazy loader. Each component is a plain `HTMLElement` subclass you import and register yourself. Preferred for consumption inside another bundler (Vite/webpack) because it tree-shakes and avoids the double-bundling problem.
- **`www`** — a deployable static site/app (used by the app starter, supports prerendering).
- **`hydrate`** — a server-side rendering/hydration build that renders components to HTML strings for SSR and static site generation.
- **`docs-readme` / `docs-json` / custom-elements manifest** — generated API docs from the decorators and JSDoc.
- **framework wrappers** — `@stencil/react-output-target`, `@stencil/vue-output-target`, `@stencil/angular-output-target` generate thin binding layers so the web components feel native in each framework's typing and event model.

Reactivity is decorator-driven: `@Prop` (public attribute/property), `@State` (internal reactive field), `@Watch` (react to prop/state change), `@Event`/`@Listen` (custom DOM events), `@Method` (async public methods), and lifecycle hooks (`componentWillLoad`, `componentDidRender`, etc.). Styling follows the Web Components model — `shadow: true` gives real Shadow DOM isolation, `scoped: true` emulates scoping without Shadow DOM, and theming is done through CSS custom properties that pierce shadow boundaries.

## Production Notes

**Pick the right output target early.** The `dist` lazy target and `dist-custom-elements` behave differently enough that switching late is painful. Lazy loading is great for a large standalone library loaded via a single script tag; it is the wrong choice when the components are imported into an app that already has a bundler, where `dist-custom-elements` tree-shakes and avoids shipping the loader twice.

**Shadow DOM styling is the recurring support ticket.** Global stylesheets and utility CSS (Tailwind, Bootstrap) do not cross into `shadow: true` components. Theming must go through CSS custom properties and `::part()`. Teams that don't plan for this end up switching components to `scoped` mode and losing real isolation.

**Framework integration, especially React, has historical sharp edges.** Before React 19, React passed everything to custom elements as attributes (strings) and could not bind rich props or listen to custom events cleanly — the `react-output-target` wrapper existed specifically to paper over this. React 19 added proper custom-element property/event support, which reduces but does not eliminate the need for wrappers when you want first-class TypeScript types.

**Decorators are the legacy TypeScript experimental flavor**, not TC39 stage-3 decorators. This is fine today because Stencil controls the compile, but it couples projects to Stencil's compiler and is a consideration for anyone imagining an eventual bundler-native migration.

**Native form participation** was historically absent — custom elements don't submit with a `<form>` for free. Form-associated custom elements via `ElementInternals` (the `formAssociated` option) are supported now, but any component that acts as an input needs explicit wiring.

**Testing is Stencil's own Jest-based runner** (spec tests plus Puppeteer-driven e2e). It is convenient but has historically pinned specific Jest and Puppeteer versions, so toolchain upgrades in the surrounding app can collide with Stencil's expectations. Newer versions have loosened this and added alternative runner support.

**SSR/hydration is real but not free.** The `hydrate` output works and powers prerendering, but Shadow DOM SSR (declarative shadow DOM) support and third-party component SSR-friendliness vary; validate hydration for interactive components rather than assuming it.

## When to Use / When Not

**Use when:**
- You are building a design system or component library consumed by more than one framework (or by teams that haven't picked one).
- You want standards-based custom elements with a small runtime and no lock-in to a specific app framework.
- You want batteries included: compiler, lazy loading, SSR/prerender, docs generation, and framework wrappers from one tool.
- You are already in the Ionic ecosystem.

**Avoid when:**
- You are building a single app in a single framework — that framework's native components are simpler and better integrated than compiled web components.
- You need full control of your bundler and build pipeline; Stencil wants to own it.
- Shadow DOM styling constraints conflict with a global utility-CSS design approach you can't change.
- You need bleeding-edge stage-3 decorators or a bundler-native, compiler-free authoring model — reach for a runtime library instead.

## Alternatives

- lit/lit — Google's runtime library for web components; no build step required, smaller conceptual surface, but you ship a (small) runtime and write more boilerplate. Use instead when you want standards components without owning the build.
- microsoft/fast — web-component library and design-system tooling behind Fluent UI. Use instead when you're in the Microsoft/Fluent design orbit.
- sveltejs/svelte — also a compiler with tiny output and can emit custom elements, but is app-framework-first rather than web-components-first. Use instead when you're building an app, not a cross-framework component library.
- vuejs/core — `defineCustomElement` compiles Vue SFCs to custom elements. Use instead when your team already writes Vue and only occasionally needs web components.
- atomicojs/atomico — small function/hooks-based web-component library. Use instead when you prefer a React-hooks-style authoring model with a minimal runtime.

## History

| Version | Date | Notes |
|---------|------|-------|
| Announced | 2017-09 | Unveiled by the Ionic team at Polymer Summit[^1]. |
| 1.0 (Stencil One) | 2019-06 | Major compiler rewrite; faster builds, lazy-load architecture[^3]. |
| 2.0 | 2020-09 | Node/target updates, refined output targets. |
| 3.0 | 2023-01 | Custom-elements output improvements, ESM-first direction. |
| 4.0 | 2023-07 | TypeScript 5, dropped legacy targets, modernized toolchain[^4]. |
| 4.43.5 | 2026-05-28 | Current 4.x point release[^5]. |

## References

[^1]: Ionic Team, "Announcing Stencil" (Polymer Summit 2017); project site. https://stenciljs.com/docs/introduction
[^2]: npm package `@stencil/core`; repository now at github.com/stenciljs/core. https://www.npmjs.com/package/@stencil/core
[^3]: Ionic Blog, "Stencil One (1.0.0)". https://ionic.io/blog/stencil-one-is-here
[^4]: Stencil documentation, upgrade guides (v4). https://stenciljs.com/docs/upgrading-to-stencil-four
[^5]: GitHub releases, stenciljs/core (v4.43.5, 2026-05-28). https://github.com/stenciljs/core/releases

## Tags

typescript, web-components, custom-elements, compiler, jsx, design-system, ssr, ssg, framework-agnostic, ionic, frontend
