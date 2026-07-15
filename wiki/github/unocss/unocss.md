# unocss/unocss

> The instant on-demand atomic CSS engine — a preset-driven alternative to Tailwind with no hardcoded core utilities.

[GitHub repo](https://github.com/unocss/unocss) ·
[Official website](https://unocss.dev) ·
[License: MIT](https://github.com/unocss/unocss/blob/main/LICENSE)

## Overview

UnoCSS is an atomic-CSS engine created by Anthony Fu (`antfu`) and first published in October 2021[^1]. Rather than shipping a fixed set of utility classes, it ships an engine: rules, variants, and shortcuts are all defined in configuration, and the utility vocabulary you actually use (Tailwind-like, Windi-like, or entirely bespoke) is supplied by *presets*. Its origin is the sunset of Windi CSS — Anthony Fu maintained Windi, and UnoCSS is its architectural successor, rebuilt around a smaller, framework-agnostic core[^2].

The defining tradeoff is **flexibility versus convention**. Because there are no built-in utilities, two UnoCSS projects with different presets can share zero class vocabulary; the "batteries" are opt-in. This buys extreme customization (define `text-primary` as a real rule, not a config override) and a very small runtime (~6kb, zero dependencies), at the cost of a steeper mental model than Tailwind's "install and use the docs" path. Teams that want a documented, stable utility set out of the box get less value from UnoCSS than teams that want to design their own atomic system.

As of 2026 it has ~18.9k stars and is actively developed, with releases landing multiple times per month[^3]. It is most visible in the Vite/Nuxt/Astro ecosystem, where its first-party integrations are strongest.

## Getting Started

```bash
npm i -D unocss
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import UnoCSS from 'unocss/vite'

export default defineConfig({
  plugins: [UnoCSS()],
})
```

```ts
// uno.config.ts
import { defineConfig, presetWind3, presetIcons } from 'unocss'

export default defineConfig({
  presets: [presetWind3(), presetIcons()],
  shortcuts: {
    'btn': 'px-4 py-1 rounded bg-blue-500 text-white hover:bg-blue-600',
  },
})
```

```ts
// main.ts — pull the generated stylesheet into your bundle
import 'virtual:uno.css'
```

```html
<button class="btn">Save</button>
<div class="i-mdi-home text-2xl" />   <!-- pure-CSS icon via preset-icons -->
```

## Architecture / How It Works

The core (`@unocss/core`) is a pure function: given a set of extracted string tokens and a config, it returns generated CSS. It has no framework dependency, no CLI, and no bundler assumptions — those live in separate integration packages. The pipeline has four stages:

1. **Extractors** — scan source files and pull *candidate* strings (class names, attributes). There is no AST parse and no full CSS parse; extraction is regex/split-based, which is where the "instant, 5× faster than Windi/Tailwind JIT" claim in the README originates[^4].
2. **Rules** — ordered `[matcher, handler]` pairs. Static rules match an exact string (`'flex' → { display: 'flex' }`); dynamic rules match a regex and compute CSS from the capture groups. First match wins, so order matters.
3. **Variants** — transform a token before it hits a rule (`hover:`, `md:`, `dark:`), rewriting the selector or wrapping it in a media/`@supports` query.
4. **Presets** — bundles of rules + variants + shortcuts + theme. `presetWind3`/`presetWind4` provide the Tailwind-compatible vocabulary; `presetMini` is the minimal subset; `presetAttributify`, `presetIcons`, `presetTypography`, `presetWebFonts` add orthogonal capabilities.

Two features sit outside the rule engine. **Shortcuts** alias one or more utilities behind a new name (statically or via regex), resolved at generation time. **Transformers** rewrite source *before* extraction: `transformer-directives` implements `@apply` and `theme()` in real CSS files, `transformer-variant-group` expands `hover:(bg-red text-white)`, and `transformer-compile-class` collapses many utilities into a single hashed class at build time.

Integrations (`unocss/vite`, `@unocss/nuxt`, `@unocss/astro`, `@unocss/webpack`, `@unocss/postcss`, `@unocss/cli`, `@unocss/runtime`) wire the engine's extract→generate loop into a host build system and expose the output as a virtual module or emitted stylesheet.

## Production Notes

- **Content detection is the top footgun.** UnoCSS only generates CSS for tokens it can *see* as literal strings in scanned files. Dynamically composed classes (`` `text-${color}-500` ``), classes in files outside the include globs, or class names arriving from a CMS produce no CSS and fail silently — the class simply has no styles. The fix is the `safelist` config, `@unocss-include` magic comments, or restructuring to full static class names. This is the single most common support issue.
- **Preset versioning churn.** The Tailwind-compatible preset has moved through names/majors: `presetUno` → `presetWind3` → `presetWind4`, with `presetWind4` reworking the theme to align with Tailwind v4 concepts. Upgrading across these is not a drop-in; class output and theme keys can shift. Pin your preset and read its migration notes before a major bump.
- **Version numbering discontinuity.** The project stayed on `0.x` for over three years, then dropped the leading zero: `0.65.4` (Jan 2025) was followed by `65.4.0`, then `66.0.0`[^5]. The large major number is a formatting change, not a signal of maturity churn — but tooling that assumed a `0.x` range in `package.json` broke on the jump.
- **Editor tooling is required for ergonomics.** Because utilities are generated on demand, the VS Code extension (or the language-server package) is what gives autocomplete and hover previews. Without it, contributors are typing against invisible, config-defined vocabulary with no IDE feedback.
- **Not a Tailwind drop-in.** `presetWind*` covers most Tailwind utilities but is not 100% identical; arbitrary-value syntax, some plugins, and config shape differ. Migrating an existing Tailwind codebase is a port, not a swap.
- **Runtime mode cost.** `@unocss/runtime` (CDN, generate-in-browser) is convenient for demos/prototypes but ships the engine to the client and generates CSS at runtime; it is not the recommended path for production apps, which should generate at build time.

## When to Use / When Not

**Use when:**
- You want to design your own atomic vocabulary (real rules, not just config theme overrides).
- You're on Vite / Nuxt / Astro and want a tiny, zero-dependency CSS layer with fast rebuilds.
- You want pure-CSS icons, attributify mode, or `@apply`/variant-group ergonomics without extra tooling.
- You're comfortable owning your preset choices and pinning versions.

**Avoid when:**
- You want the documented, stable, "just follow the Tailwind docs" experience with the largest community and plugin ecosystem.
- Your class names are heavily dynamic/runtime-composed and you don't want to maintain a safelist.
- Your team is small and won't install the editor tooling that makes on-demand utilities usable.
- You need a large body of Stack Overflow answers and third-party UI kits — Tailwind's ecosystem is far larger.

## Alternatives

- tailwindlabs/tailwindcss — use instead when you want the mainstream, best-documented utility framework with the biggest ecosystem and don't need engine-level customization.
- windicss/windicss — the predecessor; now sunset and unmaintained. Use UnoCSS instead of adopting it new.
- unjs/... (n/a) — not a substitute; listed only to note UnoCSS shares the antfu/UnJS tooling orbit.
- tw-in-js/twind — use instead when you specifically want CSS-in-JS with a runtime/SSR shim rather than build-time generation.
- css-modules/css-modules or vanilla CSS — use instead when you don't want an atomic/utility model at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.0 | 2021-10-04 | First npm publish[^1]. |
| 0.1.0 | 2021-10-26 | First tagged release. |
| 0.65.4 | 2025-01-08 | Final `0.x` release before renumbering[^5]. |
| 65.4.0 | 2025-01-10 | Dropped the leading zero; same line, new major format. |
| 66.0.0 | 2025-02-18 | Current major line; `presetWind4` / Tailwind-v4-aligned theming lands in this era. |
| 66.7.5 | 2026-07 | Latest release at time of writing[^3]. |

## References

[^1]: npm registry, `unocss` publish history — first version `0.0.0` on 2021-10-04. https://www.npmjs.com/package/unocss?activeTab=versions
[^2]: Anthony Fu, "Reimagine Atomic CSS" — design rationale behind UnoCSS. https://antfu.me/posts/reimagine-atomic-css
[^3]: GitHub repository metadata, fetched 2026-07 (stars ~18.9k, latest `v66.7.5`). https://github.com/unocss/unocss/releases
[^4]: UnoCSS README, "Features" — "No parsing, no AST, no scanning ... 5x faster than Windi CSS or Tailwind JIT". https://github.com/unocss/unocss#features
[^5]: npm version timeline — `0.65.4` (2025-01-08) → `65.4.0` (2025-01-10) → `66.0.0` (2025-02-18). https://www.npmjs.com/package/unocss?activeTab=versions

## Tags

typescript, css, atomic-css, utility-first, build-tooling, vite-plugin, tailwind-alternative, frontend, styling, preset-engine
