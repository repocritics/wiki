# streamich/react-use

react-use — the essential React hooks utility library. ~100 well-tested hooks for things React itself doesn't ship but every team needs.

## What it is

A TypeScript library of ~100 reusable React hooks covering UI state, browser APIs, lifecycle events, sensors, animation, and side-effects. Saves teams from reinventing the same `useDebounce` / `useLocalStorage` / `useMedia` 100 times. The unofficial "VueUse for React" — pioneered the model of curating a single utility-hooks library that becomes the team's default.

## Key features

- ~100 hooks organized into categories: state, side-effects, lifecycle, animation, sensors (geolocation, network, idle, hover, scroll), UI utilities (useClickAway, useKey, useWindowSize), function utilities (useDebounce, useThrottle, useUpdateEffect).
- Storage hooks: `useLocalStorage`, `useSessionStorage`, `useCookieValue`.
- Network: `useFetch`, `useAsync`, `useAsyncFn`, `useNetworkState`.
- DOM events: `useEvent`, `useClickAway`, `useKey`, `useKeyPress`.
- TypeScript types bundled.
- Tree-shakable — `import { useDebounce } from 'react-use'` only pulls what's used.
- Unlicense (public-domain-equivalent) — friction-free vendoring.

## Tech stack

- TypeScript primary.
- npm package: `react-use`.
- Hook implementations stay close to native React semantics — no opinionated frameworks layered on top.

## When to reach for it

- You're a React team and want a vetted utility-hooks library to anchor on.
- You're maintaining a codebase where homegrown hooks have proliferated and want a consolidation target.
- You want a permissive license (Unlicense) for redistribution without attribution.

## When *not* to reach for it

- You only need 1-2 hooks — copy them inline rather than adding a dependency.
- You want a smaller, modern alternative — `@uidotdev/usehooks` and `usehooks-ts` ship tighter scopes.
- You're sensitive to bundle size and don't tree-shake aggressively — verify your bundler's output before assuming it's free.

## Maturity signal

Actively maintained for 5+ years. The hook catalog is mostly stable — additions are slow, breakage is rare. Used by thousands of React projects. Unlicense + lean dependency set + stable API make this the safest utility-hooks library to depend on long-term.

## Alternatives

- `@uidotdev/usehooks`, `usehooks-ts` — alternatives with overlapping coverage and tighter scopes.
- `VueUse` for Vue.js — the spiritual sibling project.
- React Hook Form, TanStack Query — for domain-specific hook libraries (forms, server state).
- Copy-paste from `usehooks.com` — when you want one-off snippets without a dependency.

## Notes

The Unlicense is unusually permissive — most utility libraries pick MIT. For internal-only projects this doesn't matter; for libraries you redistribute, Unlicense means downstream users have zero attribution obligation. The author (streamich) maintains several other utility libraries; this one is the most-installed by far.

## Tags

react, typescript, hooks, library, utility, unlicense
