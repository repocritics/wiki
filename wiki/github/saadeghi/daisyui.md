# saadeghi/daisyui

> A Tailwind CSS plugin that adds semantic component class names (`btn`, `card`, `modal`) on top of utility classes — no JavaScript, framework-agnostic.

[GitHub repo](https://github.com/saadeghi/daisyui) ·
[Official website](https://daisyui.com) ·
[License: MIT](https://github.com/saadeghi/daisyui/blob/master/LICENSE)

## Overview

daisyUI is a component library implemented as a single Tailwind CSS plugin, created by Pouya Saadeghi and first released in 2020[^1]. Where Tailwind is deliberately utility-first — you compose `px-4 py-2 rounded bg-blue-600 …` inline — daisyUI reintroduces named component classes (`btn btn-primary`, `card`, `tabs`, `modal`) that expand into those utilities plus a theme-variable layer. It ships no JavaScript: every component is pure CSS, so it works identically in plain HTML, React, Vue, Svelte, Angular, Astro, or a server-rendered template. This is the core distinction from React-bound libraries like shadcn/ui or MUI.

The defining tension is philosophical. Tailwind's own guidance argues against semantic class names because they recreate the naming and specificity problems utility-first was meant to kill. daisyUI bets the opposite way: for the common 80% of UI (buttons, forms, cards, alerts) a short semantic class is more readable and more maintainable than a 12-utility soup, and you drop back to raw utilities for the custom 20%. Whether that abstraction is worth a leaky boundary between "daisyUI's class" and "your overrides" is the recurring debate around the project.

At ~41,700 stars, ~1,660 forks, and commits within the last day of this writing, it is the most-starred Tailwind component library and actively maintained. Open issues sit around three dozen, which for a project this size indicates tight triage rather than neglect. The GitHub "language" reads as Svelte only because the documentation site (daisyui.com) is built with it; the shipped library is CSS.

## Getting Started

daisyUI 5 targets Tailwind CSS v4 and its CSS-first configuration[^2]:

```bash
npm i -D daisyui@latest
```

```css
/* app.css */
@import "tailwindcss";
@plugin "daisyui";
```

```html
<!-- Works in any framework — these are just class names -->
<button class="btn btn-primary">Save</button>

<div class="card w-96 bg-base-100 shadow-sm">
  <div class="card-body">
    <h2 class="card-title">Account</h2>
    <p>Utilities and daisyUI classes compose freely.</p>
    <div class="card-actions justify-end">
      <button class="btn btn-ghost">Cancel</button>
    </div>
  </div>
</div>

<!-- Theme is set by a data attribute, not a JS provider -->
<html data-theme="cupcake"></html>
```

On Tailwind v3, pin daisyUI v4 and register it in `tailwind.config.js`'s `plugins` array instead — the two config styles are not interchangeable.

## Architecture / How It Works

daisyUI is a Tailwind plugin that injects three things into the generated stylesheet:

1. **Component classes** — `btn`, `card`, `input`, `tab`, etc. Each is a normal CSS rule set built from design-token variables, added at Tailwind's `components` layer so ordinary utilities can override it.
2. **A theme system** — colors are CSS custom properties (`--color-primary`, `--color-base-100`, …) scoped by a `data-theme` attribute on any ancestor element. Switching themes is a single attribute change; no re-render, no JS. daisyUI ships ~35 built-in themes (light, dark, cupcake, dracula, etc.) and lets you define your own via the plugin config.
3. **Modifier classes** — `btn-primary`, `btn-sm`, `btn-outline`, `alert-error`. These map to the same theme variables, so a component recolors automatically when the active theme changes.

Because everything resolves to CSS variables and utility classes, there is no runtime and no component state. "Interactive" widgets (dropdown, modal, drawer, accordion) are implemented with pure-CSS tricks — the checkbox/`:focus`/`:target`/`popover` patterns and `<details>`/`<dialog>` elements — rather than JavaScript. That is what keeps it framework-agnostic, but it also caps how much behavior a component can encode: anything needing real focus trapping, keyboard nav, or async state is left to you or a headless library.

In v5 the plugin was rebuilt around Tailwind v4's engine, dropping the older PostCSS/`tailwind.config.js` path in favor of the `@plugin` CSS directive and native CSS variables[^2]. The plugin footprint shrank and per-component CSS is emitted only for what Tailwind's content scan actually detects.

## Production Notes

**Specificity and override order.** daisyUI classes live in the `components` layer; Tailwind utilities live in `utilities`, which comes later and wins. Usually that is what you want (`btn` + `rounded-full` just works). But mixing a daisyUI modifier and a conflicting utility on the same property produces order-dependent results that surprise people — reach for `!` (important) sparingly and prefer utilities over fighting the component class.

**Content scanning still applies.** Component CSS is emitted only if the class appears in a scanned file. Dynamically constructed class strings (`` `btn-${variant}` ``) are invisible to the scanner and silently produce unstyled output. Safelist them or write full class names.

**Theme variables are the real API.** Customizing colors means overriding CSS variables (or defining a theme in the plugin config), not editing component classes. Teams that treat daisyUI classes as final markup and hard-code hex colors alongside them lose the automatic theming that is the library's main payoff.

**Upgrades track Tailwind, not just daisyUI.** The v4→v5 jump is really a Tailwind v3→v4 migration: CSS-first config, changed color/opacity syntax, and the `@plugin` directive. Budget for it as a Tailwind upgrade, not a drop-in bump. Renamed/removed utility color names between majors are a common source of post-upgrade visual diffs.

**Accessibility is your job past the basics.** CSS-only modals and dropdowns give you the look but not full ARIA semantics, focus management, or escape handling in every browser. For rigorous a11y, pair daisyUI's styling with a headless behavior library (Radix, Headless UI, Ark) rather than shipping the CSS-only interactive widgets as-is.

## When to Use / When Not

**Use when:**
- You already use Tailwind and want to stop hand-assembling the same button/card/form utilities.
- You need the same components across multiple frameworks or plain HTML/email/SSR templates.
- You want built-in multi-theme (light/dark and beyond) via a single attribute, with no JS theme provider.
- You value a small, dependency-free CSS layer over a full component-logic framework.

**Avoid when:**
- You need behavior-rich, fully accessible interactive components out of the box (use a headless React/Vue library plus your own or daisyUI styling).
- Your team dislikes semantic class names on principle and prefers pure utility composition or CSS-in-JS.
- You need components that own their own state and data logic (MUI, Chakra, Mantine fit that better).
- You are not using Tailwind at all — daisyUI is a Tailwind plugin, not standalone CSS.

## Alternatives

- shadcn/ui — copy-paste React + Radix components you own and edit; use it when you want the component source in your repo and real accessibility behavior, and you are React-only.
- tailwindlabs/tailwindcss (Tailwind Plus / Tailwind UI) — official first-party, professionally designed, paid component set; use it when budget is fine and you want maintainer-blessed patterns.
- themesberg/flowbite — Tailwind component library with an optional JS layer for interactivity; use it when you want richer built-in behavior than daisyUI's CSS-only widgets.
- mui/material-ui — full React component framework with logic and theming; use it when you need stateful, batteries-included components and are not committed to Tailwind.
- picocss/pico — classless/semantic-HTML CSS; use it when you are not using Tailwind and want sensible defaults with near-zero markup.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2020–2021 | Initial release as a Tailwind CSS plugin adding component classes[^1]. |
| 2.x | 2022 | Expanded component set and theming. |
| 3.x | 2023 | Theme and color-system refinements. |
| 4.x | 2023 (late) | Tailwind v3-era release; `tailwind.config.js` plugin registration. |
| 5.x | 2025 | Rebuilt for Tailwind CSS v4: CSS-first `@plugin` config, native CSS variables[^2]. |

## References

[^1]: daisyUI — GitHub repository and release history. https://github.com/saadeghi/daisyui/releases
[^2]: daisyUI 5 install and Tailwind CSS v4 configuration. https://daisyui.com/docs/install/
[^3]: daisyUI themes documentation (`data-theme`, built-in themes, custom themes). https://daisyui.com/docs/themes/

## Tags

css, tailwindcss, component-library, design-system, ui-kit, frontend, plugin, theming, framework-agnostic, svelte
