# shadcn-ui/ui

A "copy-into-your-repo" component library — beautifully designed, Radix-UI-backed React components plus a code distribution platform. Has reshaped how teams build design systems in 2024-2026.

## What it is

A React component library that breaks the "install via npm" convention: components are copied directly into your project as source code, customizable in place. Built on Radix UI primitives (for accessibility) + Tailwind CSS (for styling) + class-variance-authority (for variants). The CLI (`npx shadcn`) handles the copy-paste; you own the resulting code. Distinguishes itself from MUI / Mantine / Chakra by giving full control over component internals at the cost of "no semver upgrades" — when you copy the code, you take responsibility for maintaining it.

## Key features

- "Copy-paste" component model — components live in your codebase, not in node_modules.
- Built on Radix UI primitives (accessibility) + Tailwind (styling).
- CLI distribution: `npx shadcn add <component>` writes files into your project.
- 50+ components: forms, dialogs, dropdowns, charts, data tables, sidebars, navigation, themes.
- Theme system with CSS variables — light/dark mode, multiple color palettes shipping as presets.
- Multi-framework support: Next.js (canonical), Vite, Laravel, TanStack, others.
- "Code distribution platform" — registries can publish their own component sets via the same CLI.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- React on the UI side.
- Tailwind CSS for styling.
- Radix UI / Base UI for accessibility primitives.
- The CLI is a Node tool that fetches component source from a registry and writes to the project tree.

## When to reach for it

- You want a high-quality component baseline but resent the "npm-install a black box" pattern.
- You want full control over component internals (custom variants, brand-specific tweaks).
- You're on Next.js / Vite / Tailwind already — the path of least friction.
- You're building a design-system distribution platform yourself — the registry pattern is the differentiator.

## When *not* to reach for it

- You want npm-managed dependencies with version bumps and CHANGELOGs — shadcn-ui's "own the code" model is opposite.
- You don't use Tailwind — components are styled with Tailwind utilities; porting to another CSS approach is significant work.
- You want a framework other than React — shadcn-ui is React-first, though community ports exist for Svelte/Vue.

## Maturity signal

115k stars, 9k forks, MIT, last push 2026-06-01. 3-year-old project that defined the "copy-paste-not-npm" component model. Open-issues count of 2,005 reflects the breadth of edge cases across components and framework targets. Vercel-adjacent (shadcn now at Vercel); long-term stewardship is healthy. The recent "code distribution platform" framing extends the registry pattern beyond the original component set.

## Alternatives

- `mui/material-ui`, `mantine/mantine`, `chakra-ui/chakra-ui` — use when you want npm-managed component libraries with version bumps.
- `radix-ui/primitives` — use when you want the unstyled accessibility primitives directly without shadcn's styling layer.
- `tailwindlabs/headlessui` — use when you want unstyled components from the Tailwind team specifically.
- DaisyUI — use when you want pre-styled Tailwind components via npm install.

## Notes

The "code distribution platform" framing in the description is recent — shadcn-ui's CLI now supports custom registries, so internal company component sets can be distributed via the same mechanism. License (MIT) makes the copy-paste model legally clean. The trade-off — "no automatic upgrades" — is the project's defining choice; teams that want updates have to re-run the CLI per component and merge changes manually.

## Tags

react, typescript, tailwindcss, radix-ui, ui, components, design-system, frontend, accessibility, nextjs
