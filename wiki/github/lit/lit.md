# lit/lit

> A small library for building fast web components on browser-native standards — the successor to Google's Polymer.

[GitHub repo](https://github.com/lit/lit) ·
[Official website](https://lit.dev) ·
[License: BSD-3-Clause](https://github.com/lit/lit/blob/main/LICENSE)

## Overview

Lit is a library for authoring Web Components — custom elements that run natively in the browser without a framework runtime. It is developed by a team at Google as the successor to Polymer, and it is the foundation of many shipping design systems (Google's own Material Web, SAP UI5 Web Components, Adobe Spectrum, ArcGIS)[^1]. The core `lit` package is roughly 5–6 KB minified and gzipped, which is the number the project leads with.

Lit is composed of three layers that ship as separate packages but are usually consumed together through the umbrella `lit` package: `lit-html` (a template system built on tagged template literals), `@lit/reactive-element` (a low-level base class adding a reactive property/attribute lifecycle to `HTMLElement`), and `lit-element` / `LitElement` (the component base class that combines the two)[^2]. This monorepo also holds official add-ons — `@lit/context`, `@lit/task`, `@lit/react`, `@lit/localize` — and a `labs/` tier of experimental packages including SSR.

The defining tension is that Lit commits to the browser's own component model — custom elements, Shadow DOM, and `<template>` — rather than inventing an abstraction over it. That buys framework-agnostic, long-lived components and a tiny runtime, but it also inherits every rough edge of those platform features: Shadow DOM style scoping, form participation, SSR/hydration, and framework interop are all shaped by web-platform limitations Lit cannot paper over.

## Getting Started

```sh
npm i lit
```

```ts
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('my-counter')
export class MyCounter extends LitElement {
  static styles = css`
    button { font: inherit; padding: 0.4em 0.8em; }
  `;

  @property({ type: Number }) count = 0;

  render() {
    return html`
      <button @click=${() => this.count++}>
        clicked ${this.count} times
      </button>
    `;
  }
}
```

```html
<my-counter count="3"></my-counter>
```

Decorators are optional; the same class can be written with a static `properties` block and `customElements.define(...)` for plain JavaScript with no build step.

## Architecture / How It Works

**lit-html** renders via tagged template literals. The `html\`...\`` tag returns a `TemplateResult` — a description, not DOM. On first render, lit-html parses the static string parts once into a `<template>` element and records the positions of the dynamic expressions ("parts"). On re-render it walks only those parts and updates the DOM in place. There is no virtual DOM and no diffing of static markup; the static/dynamic split is computed once per unique template literal[^2]. Bindings are typed by position: `?disabled=${b}` sets a boolean attribute, `.value=${v}` sets a property, `@click=${fn}` adds an event listener.

**Reactive updates** come from `@lit/reactive-element`. Reactive properties are backed by accessors; a change calls `requestUpdate()`, which schedules an asynchronous, batched update on a microtask. The lifecycle is `willUpdate → update → render → updated`, with `firstUpdated` on the first cycle and a `hasChanged` hook per property. Because updates are async and batched, multiple synchronous property writes coalesce into one render.

**Shadow DOM** is the default. Each element renders into its own shadow root, and `static styles` (a `css\`...\`` tagged literal) are scoped there, deduplicated across instances via Constructable Stylesheets where supported. This is the source of both Lit's best property (real style encapsulation) and most of its friction (see Production Notes).

**Reactive Controllers** (added in Lit 2.0) are the composition primitive: a plain object with lifecycle hooks (`hostConnected`, `hostUpdate`, etc.) that attaches to an element to share stateful behavior without inheritance. `@lit/task`, `@lit/context`, and `@lit-labs/observers` are all controllers. **Directives** (`repeat`, `when`, `choose`, `guard`, `cache`, `until`, `keyed`, `ref`) extend template behavior; custom directives can hook into the part system for imperative DOM control.

## Production Notes

**Shadow DOM is the recurring operational cost.** Global stylesheets and design tokens do not cross the shadow boundary; you distribute theming with CSS Custom Properties and `::part()` / `::slotted()`, which must be designed in deliberately. Native form participation requires wiring elements up through the ElementInternals / form-associated custom element API by hand — a bare LitElement with an `<input>` inside its shadow root will not submit with a surrounding `<form>`.

**SSR is still `@lit-labs`.** `@lit-labs/ssr` renders components to Declarative Shadow DOM and there is a `@lit-labs/nextjs` integration, but server rendering and hydration have remained in the experimental labs tier far longer than most of the rest of the library, and the ergonomics are meaningfully rougher than a server-first framework. If SEO-critical SSR is a hard requirement, evaluate this carefully before committing.

**Framework interop, especially React.** React versions before 19 do not set DOM properties or listen to custom-element events idiomatically, so passing objects or subscribing to events required the `@lit/react` `createComponent` wrapper. React 19 improved native custom-element support; verify behavior against your exact React version rather than assuming.

**Decorators changed under you.** Lit supports both TypeScript's legacy `experimentalDecorators` and the TC39 standard decorators. The two require different `tsconfig` settings and are not interchangeable at the syntax edges; mixing toolchains (Babel vs tsc vs esbuild) is a common source of "decorator is not a function" errors. Lit 3 targets standard decorators as the forward path.

**Attribute vs property reflection is a frequent footgun.** Attributes are strings; properties are typed. Only primitive props round-trip through attributes cleanly, and reflection (`reflect: true`) is one-directional by design. Passing objects/arrays must go through properties, which matters at every framework boundary and in SSR serialization.

## When to Use / When Not

**Use when:**
- You are building a design system or component library meant to be consumed across React, Vue, Angular, and plain HTML.
- You want durable, standards-based components with a tiny runtime and no framework lock-in.
- You are progressively enhancing server-rendered HTML with self-contained interactive elements.
- Style encapsulation via Shadow DOM is a feature you want, not a cost you tolerate.

**Avoid when:**
- You need a full application framework (routing, data fetching, SSR-first) out of the box — Lit is a component layer, and its router/SSR pieces live in labs.
- SEO-critical server rendering is a hard, day-one requirement.
- Your team is fully committed to one framework and gains nothing from web-component portability — an idiomatic React/Vue/Svelte component will integrate more smoothly.
- Shadow DOM's scoping model actively fights your global-CSS or Tailwind-first workflow.

## Alternatives

- microsoft/fast — Microsoft's web-components stack; heavier design-system tooling, similar standards bet. Use when you want FAST's opinionated component set and adaptive theming.
- ionic-team/stencil — compiler that emits standards-based web components (and framework wrappers) from a JSX-like syntax. Use when you prefer a build-time compiler over a runtime library.
- sveltejs/svelte — compiles components to minimal JS; can emit custom elements. Use when you want the smallest output and don't need the web-component model everywhere.
- preactjs/preact — 3 KB React-compatible renderer. Use when you want React semantics at a small size rather than platform-native components.
- vuejs/vue — can build and consume custom elements. Use when you're already in the Vue ecosystem and web components are a secondary concern.

## History

| Version | Date | Notes |
|---------|------|-------|
| lit-html 1.0 | 2019 | Standalone template library extracted from Polymer[^1]. |
| LitElement 2.0 | 2019 | Component base class over lit-html. |
| Lit 2.0 | 2021 | Unified `lit` package; Reactive Controllers introduced[^3]. |
| Lit 3.0 | 2023-10 | Dropped IE11, ES2021 target, standard-decorators path, SSR advances[^4]. |

## References

[^1]: Lit documentation and project overview. https://lit.dev/docs/
[^2]: Lit docs, "Templates" and "Rendering" — lit-html template/part model. https://lit.dev/docs/templates/overview/
[^3]: Lit team, "Lit 2.0" release announcement (Reactive Controllers). https://lit.dev/blog/2021-09-21-lit-2.0/
[^4]: Lit team, "Lit 3.0" release announcement. https://lit.dev/blog/2023-10-10-lit-3.0/

## Tags

typescript, web-components, shadow-dom, custom-elements, frontend, ui-library, templating, reactive, design-systems, google
