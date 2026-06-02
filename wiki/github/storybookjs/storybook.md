# storybookjs/storybook

Storybook — a workshop for building UI components in isolation. The de facto component-development environment across React, Vue, Angular, Svelte, web components, and many more frameworks.

## What it is

A TypeScript framework that gives developers a sandbox for building, testing, and documenting UI components outside the running app. Each component gets "stories" — isolated example pages showing different prop combinations and states. Used by every major design-system team (Shopify Polaris, Atlassian Atlaskit, IBM Carbon, GitHub Primer, Adobe React Spectrum) as the canonical component-development surface. Storybook Inc. funds development; Chromatic is the commercial sister product for visual regression testing.

## Key features

- Multi-framework adapters: React, Vue 3, Angular, Svelte, Web Components, Lit, Solid, Preact, Ember, HTML.
- Stories — `.stories.tsx` files describe component states as standalone examples.
- Addon ecosystem: a11y (axe-driven accessibility tests), controls (live prop editing), viewport, theming, interactions (Testing Library + Playwright integration), docs (MDX-rendered).
- Chromatic — paid visual regression testing service that diffs Storybook output across CI runs.
- MDX docs + stories side-by-side for living component documentation.
- Storybook test runner — runs every story as a smoke test in CI.
- Component-driven development workflow: design → build in isolation → integrate into app.
- MIT-licensed.

## Tech stack

- TypeScript primary across the framework core + addons.
- Per-framework adapter packages (`@storybook/react`, `@storybook/vue3`, etc.).
- Vite + Webpack builders supported.
- Distributed via the `storybook` umbrella npm package + `@storybook/*` namespace.

## When to reach for it

- You're building or maintaining a design system / component library.
- You want visual + interaction testing of components without spinning up the full app.
- You need a documentation surface that designers + product managers + engineers all consume.
- You want to onboard new engineers — Storybook stories are often the first PR a new team member writes.

## When *not* to reach for it

- Your project has <10 reusable components — Storybook's setup overhead doesn't pay off.
- You're a one-off prototype that won't outlive the week.
- You're on a stack Storybook doesn't support (rare; the framework list is broad).

## Maturity signal

10+ years old, maintained under Storybook Inc. Storybook 7 introduced a unified architecture across frameworks; Storybook 8 added test-runner improvements + Vite-first defaults. The project has weathered multiple major frontend ecosystem shifts (Webpack → Vite, React class → hooks, etc.) without breaking the core stories format. Active contributor community + commercial backing make this the safest long-term bet for component-development infrastructure.

## Alternatives

- Ladle — Vite-only lighter alternative with faster startup.
- Histoire — Vite-flavored, Vue-first competitor.
- React Cosmos — older alternative, React-only, more developer-friendly for fast iteration.
- Documentation-only: Docusaurus + MDX (no component sandbox, just docs).

## Notes

The "stories format" (CSF — Component Story Format) has become a portable convention — even projects not using Storybook sometimes adopt the format for component examples. Chromatic's visual-regression integration is the canonical "we caught a CSS regression before merge" story across many engineering teams. The MIT license + commercial Chromatic offering is the standard healthy commercial-OSS pattern.

## Tags

typescript, react, vue, angular, ui, components, design-system, documentation, mit-license, testing
