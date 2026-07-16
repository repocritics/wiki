# storybookjs/storybook

> A frontend workshop for building, documenting, and testing UI components in isolation — outside the app that consumes them.

[GitHub repo](https://github.com/storybookjs/storybook) ·
[Official website](https://storybook.js.org) ·
[License: MIT](https://github.com/storybookjs/storybook/blob/next/LICENSE)

## Overview

Storybook is a development environment that renders individual UI components in isolation, driven by small files called *stories* that declare a component plus a set of input states[^1]. It started in 2016 as "React Storybook" (originally by Arunoda Susiripala at Kadira) and grew into a framework-agnostic tool covering React, Vue, Angular, Svelte, Web Components, Preact, and others[^2]. At ~90.7k stars and ~10.3k forks it is the de facto standard for component-driven development and design systems, and it is heavily used as the input surface for visual-regression and interaction testing.

The defining tension is scope. Storybook is not one small library; it is a monorepo of dozens of published packages — a manager UI, per-framework renderers, two build backends, and a large addon ecosystem — that must all stay in lockstep and must slot into whatever bundler and framework version your app already uses. That breadth is why it works across so many stacks, and also why major upgrades and addon compatibility are the recurring operational cost. Development is active: the `next` default branch, ~1,800 open issues, and near-daily pushes reflect a large surface under continuous maintenance rather than a settled, low-churn library.

Storybook is maintained by an open community alongside Chromatic (the company, formerly Chroma), whose hosted visual-testing product is built on Storybook output — a commercial-adjacent-to-OSS arrangement common in the JS tooling ecosystem.

## Getting Started

Add Storybook to an existing project — the CLI detects the framework and builder:

```bash
npx storybook@latest init
npm run storybook   # starts the dev server, usually on :6006
```

A story file using Component Story Format (CSF 3), the current authoring format:

```tsx
// Button.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { expect, userEvent, within } from "storybook/test";
import { Button } from "./Button";

const meta: Meta<typeof Button> = {
  title: "UI/Button",
  component: Button,
  args: { children: "Click me" },   // default args for all stories
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: { variant: "primary" },
};

// A play function turns a story into an interaction test
export const Clicked: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    await userEvent.click(canvas.getByRole("button"));
    await expect(canvas.getByRole("button")).toBeEnabled();
  },
};
```

`.storybook/main.ts` declares the story globs, addons, and framework preset; `.storybook/preview.ts` declares global decorators and parameters.

## Architecture / How It Works

Storybook runs as **two separately-built applications communicating over a channel**:

1. **The manager** — the outer Storybook UI (sidebar, toolbar, addon panels). It is a fixed app, built and shipped by Storybook itself.
2. **The preview** — an `<iframe>` that loads *your* components with *your* framework and bundler config. Isolating the preview in an iframe is what lets a component render as if it were the only thing on the page, and keeps the manager framework-neutral.

The two sides talk through an addon **channel** (postMessage across the iframe boundary). Addons are the extension model: a panel/tool on the manager side plus a preview-side hook, wired through that channel. Core addons (actions, backgrounds, viewport, measure, outline, a11y, docs, interactions) follow the same contract as third-party ones.

Two axes compose the install:

- **Renderer** — how a story becomes DOM for a given framework (`@storybook/react`, `@storybook/vue3`, `@storybook/angular`, `@storybook/svelte`, `@storybook/web-components`, …).
- **Builder** — how source is bundled: `@storybook/builder-vite` or `@storybook/builder-webpack5`. A **framework** package (e.g. `@storybook/react-vite`, `@storybook/nextjs`) bundles a renderer + builder + presets so users pick one dependency.

**Component Story Format (CSF)** is the on-disk contract: a story file is a plain ES module whose default export is metadata (`component`, `title`, `args`) and whose named exports are stories. Because CSF is standard JavaScript, the same files feed the dev UI, the docs generator, and the test runner without a bespoke DSL. CSF 3 (object stories, `args`, `play` functions) is the current form; the old imperative `storiesOf` API was removed in 8.0.

Since 7.0, story indexing is **on-demand**: the manager loads a lightweight index and only compiles a story's code when it is opened, which is what keeps large Storybooks usable. `storybook build` emits a static site (the manager plus a prebuilt preview) that can be hosted anywhere and is the artifact fed to Chromatic, the test-runner, or a CI visual diff.

## Production Notes

**Major upgrades are the main cost.** The CLI ships automigrations (`npx storybook@latest upgrade`) that rewrite config for known breaks, but each major has moved the floor: 6→7 pushed ESM + Vite as first-class and reworked the build; 8.0 removed `storiesOf` and several legacy addons; 9.0 consolidated packages and leaned hard into testing. Budget real time for majors, not a version bump.

**Addon compatibility lags core.** Third-party addons declare peer ranges against Storybook majors and frequently break the day a new major lands. Before upgrading a design system's Storybook, audit every addon's support for the target version — this is the most common upgrade blocker in practice.

**Builder choice is load-bearing.** Vite is fast and now the common default; the webpack5 builder is slower to boot and heavier, but is still required by some framework integrations and older setups. Running a Vite app with a webpack Storybook means maintaining a second, divergent build config just for stories.

**Config drift from the real app.** Because the preview is its own bundle, module resolution, aliases, env vars, PostCSS/Tailwind setup, and transpiler settings must be reproduced in `.storybook/`. Components that "work in the app but not in Storybook" (or vice-versa) are almost always this drift, not a component bug.

**Testing has absorbed the roadmap.** `play` functions run in the browser as interaction tests; the Storybook Test integration runs stories as tests under Vitest, reusing the same files. This is capable but adds a Vitest/browser-mode toolchain to maintain, and it overlaps with whatever component-testing setup a team already runs (Testing Library, Playwright/Cypress CT) — pick one story-vs-test source of truth to avoid duplicate coverage.

**React Native and native targets live in separate repos** (`storybookjs/react-native`, `storybookjs/native`) on their own release cadence; do not assume web-Storybook version parity there.

## When to Use / When Not

**Use when:**
- You are building a design system or a shared component library and need an isolated catalog of every component state.
- You want auto-generated component docs (props tables, controls) co-located with the components.
- You want stories to double as the fixtures for visual-regression and interaction testing.
- Your team spans multiple frameworks and you want one consistent workshop UI across them.

**Avoid when:**
- The project is a small app with few reusable components — the config and build overhead outweigh the benefit.
- You want a near-zero-config, lightweight story viewer — Ladle or Histoire are leaner.
- You cannot absorb periodic major-version migration and addon-compatibility churn.
- Your components are so app-context-coupled that reproducing the environment in the preview iframe costs more than it saves.

## Alternatives

- [tajo/ladle](https://github.com/tajo/ladle) — Vite-native, CSF-compatible story viewer; use when you want a lighter, faster React-only alternative with less surface area.
- [histoire-dev/histoire](https://github.com/histoire-dev/histoire) — Vite-native stories built for Vue/Svelte; use when you're all-in on Vite and want a lighter tool.
- [react-cosmos/react-cosmos](https://github.com/react-cosmos/react-cosmos) — component sandbox/fixtures for React; use when you prefer a fixtures model over Storybook's addon ecosystem.
- [styleguidist/react-styleguidist](https://github.com/styleguidist/react-styleguidist) — Markdown-driven component docs; use when documentation, not an interactive workshop, is the goal.
- [chromaui/chromatic-cli](https://github.com/chromaui/chromatic-cli) — not a replacement but the common companion; use for hosted visual regression on top of a Storybook build.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Initial release as "React Storybook"[^2]. |
| 3.0 | 2017 | Rebranded to Storybook; multi-framework direction. |
| 5.0 | 2019-04 | Unified manager UI, monorepo consolidation[^3]. |
| 6.0 | 2020-08 | Args + Component Story Format (CSF) as the authoring model[^4]. |
| 7.0 | 2023-04 | Vite first-class, on-demand story loading, refreshed docs[^5]. |
| 8.0 | 2024-03 | `storiesOf` removed, faster testing path, React Server Component support[^6]. |
| 9.0 | 2025-06 | Consolidated packages, smaller install, component testing centered on Vitest[^7]. |

## References

[^1]: Storybook documentation — "What's a story?". https://storybook.js.org/docs/get-started/whats-a-story
[^2]: Storybook documentation — Get started / overview. https://storybook.js.org/docs
[^3]: Storybook blog, "Storybook 5.0". https://storybook.js.org/blog/storybook-5-0/
[^4]: Storybook blog, "Storybook 6.0". https://storybook.js.org/blog/storybook-6-0/
[^5]: Storybook blog, "Storybook 7.0". https://storybook.js.org/blog/storybook-7-0/
[^6]: Storybook blog, "Storybook 8.0". https://storybook.js.org/blog/storybook-8-0/
[^7]: Storybook blog, "Storybook 9". https://storybook.js.org/blog/storybook-9/

## Tags

component-workshop, ui-development, design-systems, component-testing, storybook, frontend-tooling, javascript, typescript, react, vue, documentation, visual-testing
