# emotion-js/emotion

> A runtime CSS-in-JS library for React (and vanilla JS) that composes styles into hashed class names with aggressive caching.

[GitHub repo](https://github.com/emotion-js/emotion) ·
[Official website](https://emotion.sh/) ·
[License: MIT](https://github.com/emotion-js/emotion/blob/main/LICENSE)

## Overview

Emotion is a CSS-in-JS library that lets you write styles as JavaScript objects or tagged template strings and injects them into the document as real CSS rules at runtime. It ships in two shapes: a framework-agnostic core (`@emotion/css`) that returns class-name strings, and a React binding (`@emotion/react` plus `@emotion/styled`) that adds the `css` prop, theming, and a `styled` component factory closely modeled on styled-components[^1]. Since 2017 it has been one of the two dominant runtime CSS-in-JS libraries alongside styled-components, and it powers the styling layer of large component systems (Sentry, MUI's older versions, many design systems).

Emotion's design goal is predictable composition. Styles serialize into deterministic hashed class names, and later styles win over earlier ones without the specificity wars that plague hand-written CSS. It leans on heavy caching (via `@emotion/cache`, built on the `stylis` parser/prefixer) so that repeated renders of the same styles are cheap. The tradeoff is that all of this happens in the browser at render time: emotion adds runtime JavaScript, executes on every styled render, and mutates a `<style>` tag as your UI mounts.

That runtime is now the library's defining tension. React Server Components (2023 onward) cannot run emotion in server components without a `"use client"` boundary, and the broader ecosystem has shifted toward zero-runtime, build-time CSS extraction[^2]. Emotion still works well in client-rendered and traditional-SSR React apps, but it is no longer the default recommendation for greenfield RSC-first projects. Development has continued but slowed to a maintenance cadence since Emotion 11.

## Getting Started

```bash
npm install @emotion/react
# for the styled() API:
npm install @emotion/styled
```

```jsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react'
import styled from '@emotion/styled'

const box = css`
  color: hotpink;
  &:hover { color: rebeccapurple; }
`

const Button = styled.button`
  padding: 8px 16px;
  background: ${props => props.primary ? 'black' : 'white'};
`

export function App() {
  return (
    <div css={box}>
      <Button primary>Click</Button>
    </div>
  )
}
```

The `css` prop requires either the `@jsxImportSource` pragma (shown above) or the `@emotion/babel-plugin` to rewrite JSX. Without one of them the `css` prop is silently ignored.

## Architecture / How It Works

Emotion's pipeline is: **serialize → hash → insert → reference**. A `css(...)` call serializes its input (object or string) into a normalized CSS string plus a stable hash. On first use, the serialized rules are parsed and vendor-prefixed by `stylis` and inserted into a `<style>` element managed by an `@emotion/cache` instance; the element's generated class name (e.g. `css-1a2b3c`) is returned. Subsequent renders of the same styles hit the cache and only reference the existing class[^3].

The React packages layer on top of the core:

- **`@emotion/react`** provides the `css` prop, the `<Global>` component, `keyframes`, `ThemeProvider`, and the `CacheProvider` used to inject a custom cache (for nonces, `nonce`-based CSP, insertion order, or shadow DOM).
- **`@emotion/styled`** is a thin factory that reads props and theme, calls into the core serializer, and forwards a real class name to the underlying element. It reproduces the styled-components ergonomic model but on emotion's cache.
- **`@emotion/babel-plugin`** (optional) enables the `css` prop without the JSX pragma, adds source maps and human-readable `label` names in development, and can minify/hoist static styles.

The `css` prop is not native JSX. It only works because either Babel or the automatic JSX runtime rewrites elements carrying a `css` attribute into `jsx(...)` calls that emotion intercepts. This is the single most common source of "my styles don't apply" confusion, especially when mixing pragma-based and automatic-runtime files.

Server rendering is comparatively smooth: emotion 10+ supports "zero-config" SSR where styles insert automatically during `renderToString`. Streaming SSR and strict critical-CSS extraction still require `@emotion/server`'s `extractCritical` / `extractCriticalToChunks` to pull only used rules into the initial HTML[^3].

## Production Notes

**RSC / Next.js App Router.** Emotion is a client-side runtime and cannot be imported into a React Server Component. In the Next.js App Router you must mark styled files `"use client"`, and you typically need a custom cache wired through `useServerInsertedHTML` to avoid flash-of-unstyled-content on the first paint. There is no first-class App Router integration; the community patterns are workable but manual[^2]. Teams starting RSC-first apps in 2025+ frequently pick a zero-runtime library instead.

**Runtime cost.** Every styled component runs the serialize/hash path on render. For most apps this is negligible, but style-heavy lists and rapidly re-rendering trees can show measurable CPU time in the emotion serializer. Memoizing `css(...)` results outside render and avoiding prop-derived dynamic styles in hot paths are the standard mitigations.

**The `css` prop pragma trap.** Files must consistently opt into the pragma (`/** @jsxImportSource @emotion/react */`) or use the Babel plugin project-wide. Mixing conventions across files leads to styles that apply in some components and not others, with no error thrown.

**SSR class-name mismatch.** The client and server caches must produce identical class names; a mismatched `@emotion/cache` `key`, or two emotion instances from duplicate installs, causes hydration warnings and duplicated `<style>` insertion. Deduping emotion in the lockfile is a common fix.

**TypeScript theming.** The `Theme` type is `{}` by default. You must augment `@emotion/react`'s `Theme` via module declaration to get typed access to `props.theme`, or every theme access is untyped.

**Version drift.** Emotion 10 → 11 was a breaking rename (`@emotion/core` became `@emotion/react`) and changed default output; mixing 10 and 11 packages in one tree produces two runtimes and duplicated styles.

## When to Use / When Not

**Use when:**
- You're building a client-rendered or traditional-SSR React app and want colocated, composable styles with theming.
- You maintain a design system that needs dynamic, prop-driven styling and runtime theme switching.
- You want the styled-components ergonomic model with slightly better performance and caching.

**Avoid when:**
- You're building RSC-first (Next.js App Router) and want styling to work in server components without `"use client"` boundaries.
- You need zero runtime JavaScript for styling (performance budgets, static extraction, streaming to the edge).
- You want styles resolvable at build time for caching and CDN delivery — reach for a compile-time library.

## Alternatives

- styled-components/styled-components — the other major runtime CSS-in-JS with a nearly identical API; use it when your team already knows it, but it shares emotion's RSC limitations.
- vanilla-extract-css/vanilla-extract — use it when you want type-safe, zero-runtime styles extracted to static CSS files that work in RSC.
- callstack/linaria — use it when you want emotion-like syntax but compiled to static CSS at build time.
- chakra-ui/panda — use it (PandaCSS) when you want build-time atomic CSS with a design-token-first authoring model.
- facebook/stylex — use it when you want Meta's statically compiled, atomic CSS-in-JS with strong dead-code elimination.

## History

| Version | Date | Notes |
|---------|------|-------|
| 8.0 | 2018 | Early runtime API, framework-agnostic core. |
| 10.0 | 2019-02 | Major rewrite: `css` prop, `@emotion/core`, object styles, zero-config SSR[^1]. |
| 11.0 | 2020-11 | Package rename (`@emotion/core` → `@emotion/react`), improved TS types and DX[^1]. |

## References

[^1]: Emotion 11 release blog post. https://emotion.sh/docs/emotion-11
[^2]: Next.js docs, "CSS-in-JS" (App Router support caveats for runtime libraries). https://nextjs.org/docs/app/building-your-application/styling/css-in-js
[^3]: Emotion documentation — Introduction and SSR guides. https://emotion.sh/docs/introduction · https://emotion.sh/docs/ssr

## Tags

css-in-js, react, javascript, styling, css, frontend, theming, ssr, styled-components-alternative, runtime-styling
