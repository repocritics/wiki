# twbs/bootstrap

The HTML/CSS/JS framework that defined "responsive mobile-first web design" for a decade — still widely used, currently in its 5.x era with Sass-based theming and a slimmer JS payload.

## What it is

A front-end framework providing a grid system, prebuilt UI components (forms, buttons, modals, navbars, alerts, etc.), and utility classes for spacing/colors/typography. Originally built at Twitter (the `twbs` org), now community-maintained under the Bootstrap Team. The current 5.x series dropped jQuery, moved to Sass, and adopted Cosmos-style customization patterns. Documentation site at getbootstrap.com is the canonical entry point.

## Key features

- Responsive 12-column grid that anchored "mobile-first" as the default web layout convention.
- Comprehensive pre-built component set — forms, modals, dropdowns, navbars, alerts, toasts, popovers, tooltips.
- Utility-first CSS classes for layout, spacing, color, and display.
- Sass-based theming with extensive `_variables.scss` for customization.
- ESM-friendly JS modules — no jQuery dependency in 5.x.
- Long backwards-compatibility window across the 3.x / 4.x / 5.x family.

## Tech stack

- MDX as the primary language tag (documentation site source).
- Sass / SCSS for stylesheets — compiled to CSS for distribution.
- Vanilla JavaScript for component behavior; no jQuery in 5.x.
- npm packages: `bootstrap` (main), `@popperjs/core` (dropdown positioning dependency).
- MIT-licensed.

## When to reach for it

- You want a battle-tested CSS framework with broad designer/developer familiarity.
- You're building admin dashboards, marketing sites, or internal tooling where consistency matters more than visual differentiation.
- You need a framework with extensive third-party theme and template ecosystem (Bootstrap themes are the largest commercial CSS theme market).

## When *not* to reach for it

- You want utility-first CSS — Tailwind has supplanted Bootstrap in this niche.
- You need component-driven design with React/Vue framework integration — use Material UI, Mantine, shadcn/ui, etc., which design for component composition rather than HTML-class composition.
- You're optimizing for minimum CSS payload — Bootstrap's full bundle is heavy; Tailwind + tree-shaking ships much less.

## Maturity signal

174k stars, 79k forks, MIT, last push the morning this page was generated. 14-year-old project — among the oldest mainstream front-end frameworks still in active development. The fork count (79k, exceptionally high) reflects the long history of derivative themes and templates. Open-issues count of 441 is reasonable for the surface area.

## Alternatives

- `tailwindlabs/tailwindcss` — use when you want utility-first CSS with build-time tree-shaking.
- `shadcn-ui/ui` — use when you want React-component primitives that you copy into your repo.
- Bulma — use when you want a class-based framework without JS components.
- Tabler, AdminLTE — use when you want an opinionated admin-dashboard kit on top of Bootstrap.

## Notes

The 5.x jQuery-removal was the major architectural break — third-party Bootstrap themes built against 3.x or 4.x don't always port cleanly. The "Bootstrap is dead" narrative recurs in frontend discussions but the project's deployment footprint remains massive: it's still the default for back-office tooling, government sites, and educational projects. License (MIT) makes it the safest choice for clients with restrictive procurement policies.

## Tags

cascading-style-sheets, html, javascript, framework, web-framework, frontend, responsive-design, sass, user-interface
