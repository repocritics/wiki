# tailwindlabs/tailwindcss

> A utility-first CSS framework for rapidly building custom designs.

[GitHub repo](https://github.com/tailwindlabs/tailwindcss) ·
[Official website](https://tailwindcss.com) ·
[License: MIT](https://github.com/tailwindlabs/tailwindcss/blob/main/LICENSE)

## Overview

Tailwind CSS is a utility-first CSS framework: instead of writing semantic class names and corresponding stylesheets, you compose styles inline via small single-purpose classes (`flex`, `p-4`, `text-sm`, `bg-blue-500`)[^1]. The framework generates exactly the classes you use via a scanner that reads your source files. Output CSS is typically a few KB after gzip even for large applications.

Tailwind v4 (2025-01) was a near-total rewrite — the new **Oxide** engine is written in Rust, the configuration moved from `tailwind.config.js` to `@theme` blocks in CSS, the JIT engine is permanent (no more `purge` step), and PostCSS plugin authorship changed[^2]. Performance went from acceptable to dramatically faster: full builds drop from seconds to tens of milliseconds on large codebases.

The project is opinionated and contested. The utility-first philosophy is anti-idiomatic for developers trained on BEM or CSS Modules, and the inline-class approach is sometimes criticized as inline styles with extra steps. The counter-argument — that utility composition produces better long-term maintainability than custom class proliferation — has been the dominant view among adopters since ~2020.

## Getting Started

```bash
npm install tailwindcss @tailwindcss/postcss
```

`postcss.config.mjs`:

```js
export default {
  plugins: { '@tailwindcss/postcss': {} }
}
```

`src/app.css`:

```css
@import "tailwindcss";

@theme {
  --color-brand: oklch(62% 0.18 250);
}
```

Use in markup:

```html
<button class="rounded-md bg-brand px-4 py-2 text-sm font-medium text-white hover:opacity-90">
  Save
</button>
```

For Vite / Next.js / SvelteKit, framework-specific plugins exist (`@tailwindcss/vite`). The `tailwind.config.js` JavaScript config still works for backwards compatibility but is no longer the recommended path.

## Architecture / How It Works

**The Oxide engine (v4)** is a Rust binary that parses CSS, scans source files for class candidates, and emits the matching CSS rules. It replaces the JavaScript-based engine of v3, which itself replaced the AOT generator of v2. The compilation pipeline:

1. Scan source files (`*.html`, `*.jsx`, `*.svelte`, etc.) for class candidates using a regex-like scanner with content-detection heuristics.
2. Parse `@theme` blocks and any layer directives (`@layer base`, `@layer components`).
3. Generate CSS rules for matched candidates, applying variants (`hover:`, `md:`, `dark:`).
4. Emit final CSS with proper cascade order: base → components → utilities.

**Variants** (`hover:`, `focus:`, `dark:`, `md:`, `lg:`, `aria-checked:`, `data-state-open:`) compose into class names by prefix. Custom variants are declared in CSS via `@variant`.

**Theme via CSS custom properties.** v4 generates CSS variables for every theme value (`--color-blue-500`, `--spacing-4`). This means runtime theme switching is possible without rebuilds; v3 required rebuilds for theme changes.

**No more `safelist` for dynamic classes.** Class names must be statically detectable in source files. Constructing class names by string concatenation (`bg-${color}-500`) still fails — use full class names in conditionals or the `cn`/`clsx` helper pattern with explicit literals.

## Production Notes

**v3 → v4 migration.** Run `npx @tailwindcss/upgrade`[^3]. Common manual fixes:
- `tailwind.config.js` content moves to `@theme` in CSS or `@config` directive for compatibility.
- `@apply` semantics changed slightly — must be inside a layer.
- Custom plugins written for v3's JS API need rewriting against the v4 plugin API.
- PostCSS plugin moved from `tailwindcss` to `@tailwindcss/postcss`.

**Bundle size.** Production CSS is small (typically 10–50 KB before gzip for medium apps), but **arbitrary values** (`w-[342px]`) and **dynamic variants** can balloon bundle size if used carelessly. Audit with the build output report.

**IDE support.** The `Tailwind CSS IntelliSense` VSCode extension is effectively mandatory — without it, the long class strings become unreadable. The extension provides hover documentation, completion, and color previews. Treat IDE setup as part of project onboarding.

**Editor lint noise.** `eslint-plugin-tailwindcss` (community) enforces class ordering. v4's official `prettier-plugin-tailwindcss` handles ordering automatically and is the preferred path.

**Inline class noise.** Long inline class strings are the most common complaint. Mitigations: extract repeated patterns into component-level abstractions (React/Vue/Svelte components), use `@apply` sparingly for genuinely reusable patterns, use `tailwind-variants` or `cva` (class-variance-authority) for variant systems.

**Production gotchas.**
- HTML email templates: Tailwind's reset is incompatible with email clients. Use a separate email-specific config.
- Print stylesheets: most utilities don't have print variants by default; explicit `print:` variant needed.
- RTL: `rtl:`/`ltr:` variants and logical properties (`ms-4` for margin-inline-start) are supported but require attention.

## When to Use / When Not

**Use when:**
- You're building a product UI with consistent design tokens.
- Your team has bought into utility-first or is willing to learn.
- You want rapid prototyping with minimal context-switching between CSS and markup.
- You want predictable CSS bundle size that scales with usage, not codebase size.

**Avoid when:**
- You're building a content site where semantic HTML + a hand-written stylesheet is small enough.
- Your designers hand off pixel-perfect mockups in Figma with custom spacing that doesn't fit Tailwind's scale (workable but high friction).
- Your team has strong opinions against utility-first and you'd be fighting them.
- You need server-side runtime class generation that isn't in source files.

## Alternatives

- **CSS Modules** — scoped class names with normal CSS authoring. Less ecosystem leverage, no design token system.
- **vanilla-extract** — TypeScript-typed CSS-in-JS with zero runtime. Different tradeoff: types over utility composition.
- **Panda CSS** (Chakra team) — atomic CSS with type-safe recipes, closer to Tailwind v4 in philosophy.
- **UnoCSS** — Tailwind-compatible engine with faster compilation and more customization. Most Tailwind classes work; some don't.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017-11 | Initial Adam Wathan release. |
| 1.0 | 2019-05 | Stable, configurable design system. |
| 2.0 | 2020-11 | Dark mode, JIT mode preview. |
| 3.0 | 2021-12 | JIT default, arbitrary values, plugin API stable. |
| 3.3 | 2023-03 | ESM/TS config support, logical properties. |
| 3.4 | 2023-12 | Subgrid, `:has()` variant, dynamic viewport units. |
| 4.0 | 2025-01 | Oxide engine, CSS-first config, `@theme` blocks[^2]. |
| 4.1 | 2025-04 | Cascade layer improvements, container query refinements. |

## References

[^1]: Adam Wathan, "CSS Utility Classes and 'Separation of Concerns'" — 2017-08. https://adamwathan.me/css-utility-classes-and-separation-of-concerns/
[^2]: Tailwind Team, "Tailwind CSS v4.0" — 2025-01-22. https://tailwindcss.com/blog/tailwindcss-v4
[^3]: Tailwind docs, "Upgrade guide". https://tailwindcss.com/docs/upgrade-guide
