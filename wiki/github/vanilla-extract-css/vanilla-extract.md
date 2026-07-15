# vanilla-extract-css/vanilla-extract

> Write CSS in TypeScript files, get static `.css` out at build time — with typed theme variables and locally scoped class names, and nothing shipped to the browser.

[GitHub repo](https://github.com/vanilla-extract-css/vanilla-extract) ·
[Official website](https://vanilla-extract.style) ·
[License: MIT](https://github.com/vanilla-extract-css/vanilla-extract/blob/master/LICENSE)

## Overview

vanilla-extract is a zero-runtime CSS-in-TypeScript library from SEEK's open-source group[^1]. You author styles in `.css.ts` files using ordinary TypeScript objects, a bundler plugin evaluates those files at build time, and the result is a static stylesheet plus a set of hashed class-name strings that your components import. None of the styling code reaches the browser — there is no runtime style engine, no serialization on render, and no client-side injection. Conceptually it is "CSS Modules in TypeScript" extended with a typed CSS-variable theme system, `@keyframes`/`@font-face` helpers, and `calc` utilities[^2].

Its lineage runs through `treat`, an earlier SEEK CSS-in-JS library, which itself grew out of a `css-in-js-loader` experiment[^1]. The design goal is to keep the abstraction thin: what you write is standard CSS properties in an object, not a bespoke DSL. That is also the defining tension. Because evaluation happens in Node at build time, styles cannot depend on runtime values — a color that comes from props or user state cannot be baked into the extracted CSS. The library's answer is CSS custom properties (variables) plus a small optional runtime (`@vanilla-extract/dynamic`) for setting those variables inline. This split — static structure, dynamic values through variables only — is the single most important thing to internalize before adopting it.

The project is actively maintained, with commits landing through mid-2026, ~10.4k stars, and a modest but real contributor base[^3]. It sits in the "build-time CSS-in-JS" category alongside Linaria, Panda CSS, and StyleX, and against runtime libraries (Emotion, styled-components) whose approach it explicitly rejects.

## Getting Started

vanilla-extract cannot run standalone — it requires a bundler integration. Install the core package plus the plugin for your build tool:

```bash
npm install @vanilla-extract/css
npm install --save-dev @vanilla-extract/vite-plugin   # or webpack/esbuild/rollup/next/parcel plugin
```

```ts
// styles.css.ts  — evaluated at build time, produces static CSS
import { createTheme, style } from '@vanilla-extract/css';

export const [themeClass, vars] = createTheme({
  color: { brand: 'blue' },
  space: { small: '4px', medium: '8px' },
});

export const button = style({
  backgroundColor: vars.color.brand,
  padding: vars.space.medium,
  color: 'white',
  selectors: {
    '&:hover': { opacity: 0.9 },
  },
});
```

```ts
// component.ts — imports resolve to plain class-name strings
import { themeClass, button } from './styles.css';

el.innerHTML = `<div class="${themeClass}"><button class="${button}">Go</button></div>`;
```

```ts
// vite.config.ts
import { vanillaExtractPlugin } from '@vanilla-extract/vite-plugin';
export default { plugins: [vanillaExtractPlugin()] };
```

## Architecture / How It Works

The core package (`@vanilla-extract/css`) is a thin API — `style`, `globalStyle`, `createTheme`, `createThemeContract`, `keyframes`, `fontFace`, `styleVariants`, `createVar`, `fallbackVar`. Calling `style(...)` registers a rule in a per-file buffer and returns a deterministic class name derived from the file path and export identifier. The real work is done by the **integration layer** plus a bundler plugin.

When the bundler encounters a `.css.ts` file, the plugin hands it to `@vanilla-extract/integration`, which compiles and **executes the module in a Node context** (the Vite plugin uses Vite's module runner; the webpack/esbuild plugins compile then evaluate). Execution collects every registered rule, serializes it to real CSS via `@vanilla-extract/css`'s adapter, and the plugin emits that CSS as a virtual asset. The original module is then rewritten so its exports become the class-name and variable-reference strings only — all the styling logic is discarded from the client bundle. This is why the library is "zero-runtime": the browser receives a normal stylesheet and string constants, nothing else.

Themes are the second pillar. `createTheme` emits a class that sets a block of CSS custom properties and returns a typed `vars` object whose values are `var(--…)` references. `createThemeContract` defines the *shape* without values, so multiple themes can implement the same contract and be swapped by toggling a class — enabling simultaneous/nested themes without global collisions. Everything is a CSS variable under the hood, which is what makes runtime theming possible without a runtime style engine: `@vanilla-extract/dynamic`'s `assignInlineVars` just writes `style="--x: …"` on an element.

Higher-level packages build on this base: **Sprinkles** (`@vanilla-extract/sprinkles`) generates an atomic-CSS utility set from a config at build time and gives you a typed function to compose those classes — Tailwind-like ergonomics with your own design tokens. **Recipes** (`@vanilla-extract/recipes`) provides variant-based component styling (base + variants + compound variants), conceptually similar to `cva`. Both compile to plain vanilla-extract calls.

## Production Notes

- **Build-time evaluation is the whole ballgame.** `.css.ts` files run in Node, not the browser. You cannot reference `window`, DOM APIs, or any value only known at runtime. Importing app code with side effects into a `.css.ts` file can pull that code into the build-time evaluation graph and cause confusing failures. Keep `.css.ts` files to styling logic and pure constants.
- **Dynamic styles must go through variables.** Anything that changes per render (theme from user prefs, a color from a CMS, a computed width) has to be expressed as a CSS variable set via `assignInlineVars` or `setElementVars`. There is no `style({ color: props.color })` — that value doesn't exist at build time. Teams migrating from styled-components consistently hit this first.
- **Next.js App Router / Turbopack friction.** vanilla-extract's integration relies on evaluating modules during the build; this has repeatedly been a rough edge with the Next App Router and RSC, and Turbopack support has lagged behind webpack[^4]. Verify current plugin compatibility against your exact Next version before committing — this is the most common real-world adoption blocker.
- **Sprinkles output scales with your config.** Atomic classes are generated as the cartesian product of the properties and scales you declare (including responsive/conditional variants). Large, unpruned scales can produce a lot of CSS. Keep the token space deliberate; only declare conditions you use.
- **Class names are hashed, not readable.** Generated identifiers are hashes by default. Set `identifiers: 'debug'` in the plugin for development to get file/export-derived names in DevTools; the short hashes are what ship to production.
- **SSR is trivial — that's the payoff.** Because output is static CSS files plus string constants, server rendering needs no style-extraction dance (no `renderStylesToString`, no context provider). This is a genuine advantage over runtime CSS-in-JS in SSR/streaming setups.
- **Theme contracts are exhaustive by construction.** A theme implementing a contract must supply every key; missing keys are type errors. This is a feature for consistency but means large design systems front-load the token schema work.

## When to Use / When Not

**Use when:**
- You want type-safe styles and a real theming system but refuse to pay a client-side runtime cost.
- You do SSR/SSG and want to avoid runtime style-extraction complexity.
- You have a design-token system and want it enforced at the type level (contracts + Sprinkles).
- Your styling is mostly static structure with dynamic *values*, not dynamic *rules*.

**Avoid when:**
- Your styling is heavily runtime-driven in ways that don't reduce to CSS variables.
- You need a build with zero bundler-plugin configuration, or your bundler/framework combo (e.g. some Next App Router / Turbopack setups) isn't cleanly supported yet.
- You want utility-first classes without maintaining your own scale config — Tailwind is the lower-friction path.
- Your team wants to write CSS in `.css` files with minimal JS involvement — CSS Modules is simpler.

## Alternatives

- css-modules/css-modules — simpler locally-scoped CSS in plain `.css`; no type safety, theme contracts, or variable system. Use when you want scoping and little else.
- callstack/linaria — zero-runtime CSS-in-JS with tagged-template syntax; use when you prefer writing literal CSS strings over TypeScript objects.
- chakra-ui/panda — build-time CSS-in-JS with recipes and tokens, newer and framework-integrated; use when you want a more batteries-included design-system toolchain.
- facebookexperimental/stylex — Meta's atomic zero-runtime styling; use at very large scale where atomic dedup and strict determinism matter most.
- emotion-js/emotion — runtime CSS-in-JS with fully dynamic per-render styles; use when runtime cost is acceptable and props-driven rules are unavoidable.
- tailwindlabs/tailwindcss — utility-first CSS; use when you want a ready-made scale and no per-project token schema work.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | pre-2021 | Precursors `treat` and `css-in-js-loader` at SEEK establish the CSS-in-TS approach[^1]. |
| Open-sourced | 2021-03 | Repository created; first public releases of `@vanilla-extract/css`[^3]. |
| Sprinkles | 2021 | Zero-runtime atomic-CSS framework added on top of the core[^2]. |
| Recipes | 2022 | Variant-based component styling package introduced[^2]. |
| Active | 2026-07 | Ongoing maintenance; commits through mid-2026, MIT-licensed[^3]. |

## References

[^1]: README — Thanks/lineage section crediting `css-in-js-loader`, `treat`, and SEEK. https://github.com/vanilla-extract-css/vanilla-extract#thanks
[^2]: Official documentation — API, Sprinkles, and Recipes package docs. https://vanilla-extract.style
[^3]: GitHub REST API `repos/vanilla-extract-css/vanilla-extract` — created 2021-03-26, ~10.4k stars, 351 forks, MIT, last push 2026-07-14 (fetched 2026-07-15).
[^4]: vanilla-extract issues on Next.js App Router / Turbopack integration. https://github.com/vanilla-extract-css/vanilla-extract/issues

## Tags

css-in-js, zero-runtime, typescript, css-modules, styling, theming, atomic-css, build-time, design-tokens, frontend
