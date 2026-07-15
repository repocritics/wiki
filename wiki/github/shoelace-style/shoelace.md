# shoelace-style/shoelace

> A framework-agnostic library of web components built on Lit — now sunset in favor of its successor, Web Awesome.

[GitHub repo](https://github.com/shoelace-style/shoelace) ·
[Documentation](https://shoelace.style) ·
[Successor: Web Awesome](https://webawesome.com) ·
[License: MIT](https://github.com/shoelace-style/shoelace/blob/next/LICENSE.md)

## Overview

Shoelace is a library of ~55 UI components (buttons, dialogs, dropdowns, form controls, tabs, trees) shipped as standard [custom elements](https://developer.mozilla.org/en-US/docs/Web/API/Web_components) rather than as a React/Vue/Angular component set. Because each component is a real HTML element registered under the `sl-` prefix, the library works in any framework or in plain HTML with no build step, loaded straight from a CDN. It was created by Cory LaViska; the 2.x rewrite was backed by Font Awesome, where LaViska works[^1].

The project is **archived and sunset as of 2026**. The README directs all issues, pull requests, and feature requests to [Web Awesome](https://github.com/shoelace-style/webawesome), the successor built by the same team, and points existing users at a migration guide[^2]. The published `@shoelace-style/shoelace` package remains on npm under MIT for existing use, but there is no active development, and — importantly for anyone weighing adoption — **no ongoing security patching**. New work should target Web Awesome; this page documents Shoelace as it stands for teams already depending on it.

The defining tradeoff is the one inherent to web components: shadow-DOM encapsulation makes each component style-isolated and portable across frameworks, but that same isolation is why global stylesheets cannot reach inside a component. Customization happens only through the seams the author exposed — CSS custom properties, `::part()`, and slots — not through arbitrary descendant selectors. Teams that expect to style components like ordinary DOM find this the central friction.

## Getting Started

Via CDN (no build step; the autoloader registers components on first use):

```html
<link rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/@shoelace-style/shoelace@2/cdn/themes/light.css" />
<script type="module"
  src="https://cdn.jsdelivr.net/npm/@shoelace-style/shoelace@2/cdn/shoelace-autoloader.js"></script>

<sl-button variant="primary">Click me</sl-button>
<sl-rating></sl-rating>
```

Via npm, with an explicit base path for the icon assets:

```bash
npm install @shoelace-style/shoelace
```

```js
import '@shoelace-style/shoelace/dist/themes/light.css';
import { setBasePath } from '@shoelace-style/shoelace/dist/utilities/base-path.js';
import '@shoelace-style/shoelace/dist/components/button/button.js';

// Point the icon loader at the copied node_modules assets
setBasePath('/shoelace-assets');
```

## Architecture / How It Works

Components are built on [Lit](https://lit.dev)'s `LitElement` base class (earlier 2.0 betas used Stencil before the migration to Lit). Each compiles to a self-contained custom element with its own shadow root, so markup, styles, and reactive rendering are encapsulated per component. The distribution build is a custom script bundled with esbuild — there is no webpack/Rollup config to inherit.

- **Theming and customization.** A design-token layer of CSS custom properties (`--sl-color-primary-600`, `--sl-spacing-medium`, `--sl-input-height-*`) drives appearance; overriding a token on `:root` restyles every component. For fine-grained changes you reach into shadow DOM through `::part(base)` and named `slot`s. There is no way to select arbitrary internal nodes — the exposed parts are the contract.
- **Registration.** Components can be manually imported one at a time (best for bundled apps and tree-shaking) or loaded lazily by the autoloader, which watches the DOM and registers a component the first time its tag appears. The autoloader is convenient but pulls each component over the network on demand.
- **Icons.** The `<sl-icon>` system fetches SVGs at runtime relative to a base path, which is why bundled setups must call `setBasePath()` — the single most common first-run misconfiguration.
- **Forms.** Form controls are form-associated custom elements using `ElementInternals`, so `<sl-input>` and friends participate in native `<form>` submission and constraint validation without a hidden `<input>` shim.
- **Framework interop.** Vue, Angular, and Svelte consume the custom elements directly. React before v19 could not pass non-string props or bind custom events to custom elements, so Shoelace ships generated React wrappers under `@shoelace-style/shoelace/dist/react`. React 19's native custom-element support makes the wrappers largely unnecessary going forward.

## Production Notes

**FOUCE.** Until a component's module has registered, `<sl-button>` is an unknown element and renders unstyled/collapsed — a flash of undefined custom elements. The standard mitigation is a `:not(:defined) { visibility: hidden }` rule (or the `whenDefined` promise) while registration completes; this matters most with the autoloader, where registration is asynchronous.

**Icon base path.** Forgetting `setBasePath()` (or not copying `node_modules/@shoelace-style/shoelace/dist/assets` into your output) yields broken icons and 404s in production but not always in dev. Bundle the assets and set the path explicitly.

**SSR is not a happy path.** Shadow DOM plus runtime icon fetching do not server-render cleanly. Declarative Shadow DOM support across meta-frameworks is uneven, and hydration mismatches are common; most teams render Shoelace client-side only and accept the FOUCE handling above.

**Bundle size and tree-shaking.** Importing the barrel (`shoelace.js`) pulls the whole library. Cherry-pick per-component imports for production bundles; the autoloader trades bundle size for per-component network requests, which is fine on HTTP/2 but adds latency on cold loads.

**React <19 event binding.** Even with wrappers, custom-event names and imperative methods differ from idiomatic React. On React 19 you can drop the wrappers, but audit event handlers during the switch.

**Sunset risk.** The largest operational caveat is now the project's status: archived, MIT-licensed, but frozen. No new components, no bug fixes, no CVE response. Treat the current version as the terminal release and plan a migration path (Web Awesome, or vendoring a private fork) rather than expecting upstream maintenance.

## When to Use / When Not

**Use when:**
- You need a framework-agnostic component set usable across React, Vue, Svelte, and plain HTML from one dependency.
- You want drop-in components with a coherent design-token theme and a built-in dark mode, with no build step required.
- You are already on Shoelace and need to keep an existing app running.

**Avoid when:**
- You are starting a new project — adopt Web Awesome (or another maintained set) instead, since Shoelace is frozen.
- Server-side rendering / streaming HTML is a hard requirement.
- Your design system needs to deeply restyle component internals beyond the exposed parts and custom properties.
- You are all-in on a single framework and would prefer components that render without shadow-DOM and SSR caveats (e.g. Radix on React).

## Alternatives

- shoelace-style/webawesome — the official successor by the same team; the intended migration target for all new work.
- microsoft/fast — Microsoft's web-components library and design-system tooling; use when you want an enterprise-backed component base with theming primitives.
- material-components/material-web — Google's Lit-based Material Design 3 web components; use when you specifically want Material styling.
- adobe/spectrum-web-components — Adobe's Spectrum design system as web components; use inside Adobe-aligned design constraints.
- ionic-team/stencil — a compiler, not a component set; use when you want to author your own framework-agnostic components rather than consume a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2017 | Original Shoelace: a lightweight CSS-only framework (Sass), not web components. |
| 2.0 beta | ~2020 | Ground-up rewrite as web components (initially Stencil, then migrated to Lit)[^1]. |
| 2.0 | 2022 | Stable release of the Lit-based component library; Font Awesome backing. |
| — | 2024 | Web Awesome announced/crowdfunded by the Font Awesome team as the successor[^2]. |
| Sunset | 2026 | Development moved to Web Awesome; repository archived, README redirects issues/PRs[^2]. |

## References

[^1]: Shoelace README, "What are you using to build Shoelace?" — components built with LitElement, bundled with esbuild. https://github.com/shoelace-style/shoelace
[^2]: Shoelace README sunset notice and migration guidance. https://github.com/shoelace-style/shoelace and https://webawesome.com/docs/resources/migrating-from-shoelace

## Tags

web-components, custom-elements, lit, ui-library, design-system, shadow-dom, typescript, framework-agnostic, css-custom-properties, frontend, archived
