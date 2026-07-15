# callstack/linaria

> Zero-runtime CSS-in-JS: styles are written in tagged template literals and extracted to static `.css` at build time, so nothing ships to the browser at runtime.

[GitHub repo](https://github.com/callstack/linaria) ·
[Official website](https://linaria.dev) ·
[License: MIT](https://github.com/callstack/linaria/blob/master/LICENSE)

## Overview

Linaria is a CSS-in-JS library maintained by Callstack. You write styles with a familiar `css` / `styled` template-literal API, but instead of a runtime style engine injecting rules into the DOM, a build-time step evaluates the tagged templates and extracts them to real CSS files[^1]. For static styles the JavaScript payload is genuinely zero — no runtime library, no per-render style insertion. Dynamic values (e.g. `props => props.color`) are compiled to CSS custom properties, which keeps them out of the runtime as well at the cost of dropping IE11 support[^2].

The defining tension is that "zero runtime" is not free — it moves the cost to the build. To resolve interpolations like `${modularScale(2)}`, Linaria must actually execute the relevant parts of your module graph in a Node sandbox during compilation. That evaluation model is the source of both its power (arbitrary JS in styles, no CSS preprocessor needed) and its recurring pain (slow builds, cache-invalidation storms, breakage when style-source modules have side effects)[^3].

As of Linaria 5+, the build-time evaluation core was factored out of the repo into a framework-agnostic engine, `wyw-in-js`, so the same extraction machinery can back other tools[^4]. Linaria itself is now the React/DOM-facing layer (`@linaria/core`, `@linaria/react`, `@linaria/atomic`) on top of `@wyw-in-js/*`. Configuration and bundler setup largely live on the wyw-in-js side.

## Getting Started

```sh
npm install @linaria/core @linaria/react @wyw-in-js/babel-preset
```

Bundler wiring (webpack, Vite, Rollup, esbuild, Svelte) is configured through wyw-in-js, not a Linaria-specific plugin[^5].

```js
import { css } from '@linaria/core';
import { styled } from '@linaria/react';

// Framework-agnostic: `css` returns a class name string.
const header = css`
  text-transform: uppercase;
  font-size: 2rem;
`;

// React: dynamic props become CSS custom properties, resolved at runtime by the browser.
const Box = styled.div`
  color: ${props => props.color};
  &:hover { border-color: blue; }
`;

// <h1 className={header}>…</h1>
// <Box color="#333" />
```

Interpolations that reference imported values or functions are evaluated at build time; those must resolve to something serializable into CSS.

## Architecture / How It Works

The pipeline runs entirely at compile time, driven by a Babel/wyw preset that transforms each source file:

1. **Detection** — the transform finds `css` and `styled` tagged templates in a module.
2. **Shaking** — for each template, it computes the minimal slice of the module (and its imports) needed to evaluate the interpolations, discarding unrelated code. This "shaker" step exists because evaluating the whole module would run far more code than necessary[^3].
3. **Evaluation** — the shaken code is executed in a sandboxed Node context to turn `${...}` expressions into concrete strings. Dynamic-per-render values (React prop functions) are instead replaced with CSS variable references.
4. **Extraction** — the resolved rules are emitted as CSS that the bundler picks up as a virtual asset and writes to a `.css` file. `stylis` handles nesting/vendor prefixing[^1].

Because evaluation means *actually running your code*, the correctness of the output depends on style-source modules being pure. Linaria 8 (2026) requires Node.js `>=22.12.0` and moves to `wyw-in-js` v2, which defaults to a **hybrid** eval strategy: statically provable values are resolved via static analysis (Oxc-based) and only genuinely dynamic values fall back to the full evaluator[^6]. This reduces how much of your graph must be executed, but does not remove the model.

`@linaria/atomic` provides an atomic-CSS output mode (one class per declaration, deduplicated), and `@linaria/server` supports critical-CSS extraction for SSR.

## Production Notes

- **Build cost is the real price.** On large apps the build-time evaluator can dominate CI time and produce cache-invalidation storms where an edit forces re-evaluation of many modules. The project's own Stability page treats slow builds and "unexpected code executed during the build" as the common failure class, not an edge case[^3].
- **Style-source modules must be side-effect-free.** If a file used in a `css`/`styled` interpolation (or anything it imports) has side effects, they will run during the build. The documented recommendation is to move shared config and helpers into pure files[^2].
- **`css` tag is static-only.** Dynamic interpolation is supported in `styled` (via CSS variables) but not in the plain `css` tag; mixing them up is a frequent first-week surprise[^2].
- **No IE11 with dynamic styles**, because dynamic values compile to CSS custom properties[^2].
- **Config moved to wyw-in-js.** Since the engine split, `@linaria/babel-preset` gave way to `@wyw-in-js/babel-preset` and bundler docs live on wyw-in-js.dev — older tutorials referencing the Linaria-only preset are stale[^4][^5].
- **Node/toolchain floor keeps rising.** Linaria 8 requires Node 22.12+ and pulls in the Oxc dependency graph; upgrading from 6/7 is not a drop-in bump for pinned CI images[^6].
- **CSS rule ordering** can differ from source order and from runtime libraries; teams relying on exact cascade ties should verify output rather than assume.
- **Payoff is genuine.** For static-heavy UIs the runtime saving is real — Airbnb publicly documented adopting Linaria for both DX and web-performance reasons[^7].

## When to Use / When Not

**Use when:**
- You want CSS-in-JS ergonomics but refuse to ship a runtime style engine.
- Most of your styles are static or driven by a bounded set of dynamic props.
- You control your build and can keep style-source modules pure.
- You value writing logic in JS at build time over adding a CSS preprocessor.

**Avoid when:**
- Build time is already a bottleneck and you can't absorb an evaluation pass.
- Your styling is heavily runtime-dynamic (theme switching per interaction, fully data-driven rules) — a runtime library fits better.
- You need styles that are type-checked without executing modules — prefer a static, type-first approach.
- You want a minimal, low-magic setup and CSS Modules would already suffice.

## Alternatives

- vanilla-extract-css/vanilla-extract — zero-runtime and type-safe via `.css.ts` files; use it when you want compile-time styles without evaluating arbitrary module code.
- emotion-js/emotion — runtime CSS-in-JS; use it when styling must be fully dynamic at runtime and the runtime cost is acceptable.
- styled-components/styled-components — the most familiar runtime `styled` API; use it when ecosystem/DX outweigh zero-runtime goals.
- chakra-ui/panda — build-time, token/config-first styling; use it when you want design-token discipline with less module-evaluation risk.
- css-modules/css-modules — the plainest zero-runtime option; use it when you don't need JavaScript in your styles at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-05 | First release; early Linaria shipped with a small runtime[^1]. |
| 1.0 | 2019-02-07 | Zero-runtime rewrite — styles extracted at build time. |
| 2.0 | 2020-10-26 | Major release; evaluation and API refinements. |
| 4.0 | 2022-07-24 | Monorepo with scoped `@linaria/*` packages (`core`, `react`, `atomic`). |
| 5.0 | 2023-09-23 | Evaluation core factored toward the shared `wyw-in-js` engine[^4]. |
| 6.0 | 2023-12-07 | Built on `@wyw-in-js/*`; config surface migrates off `@linaria/babel-preset`. |
| 7.0 | 2026-01-26 | Major release. |
| 8.0 | 2026-06-14 | Node 22.12+, `wyw-in-js` v2, Oxc-based hybrid eval strategy[^6]. |

## References

[^1]: Linaria README and "How it works" docs, callstack/linaria. https://github.com/callstack/linaria
[^2]: Linaria README, "Trade-offs" and "Syntax" sections. https://github.com/callstack/linaria#trade-offs
[^3]: Linaria README, "Stability"; wyw-in-js stability guidance. https://wyw-in-js.dev/stability
[^4]: wyw-in-js — the framework-agnostic build-time evaluation engine spun out of Linaria. https://wyw-in-js.dev/
[^5]: wyw-in-js bundler guides (webpack, esbuild, Rollup, Vite, Svelte). https://wyw-in-js.dev/bundlers/webpack
[^6]: Linaria 8 migration notes (Node `>=22.12.0`, WyW 2, Oxc, `eval.strategy: "hybrid"`). https://github.com/callstack/linaria/blob/master/docs/MIGRATION_GUIDE.md
[^7]: "Airbnb's Trip to Linaria" — Airbnb Engineering. https://medium.com/airbnb-engineering/airbnbs-trip-to-linaria-dc169230bd12

## Tags

css-in-js, zero-runtime, css, react, typescript, build-time, styling, babel, wyw-in-js, atomic-css, callstack
