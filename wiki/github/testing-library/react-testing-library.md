# testing-library/react-testing-library

> React DOM testing utilities that test components the way users see them — the library that killed Enzyme.

[GitHub repo](https://github.com/testing-library/react-testing-library) ·
[Official website](https://testing-library.com/react) ·
[License: MIT](https://github.com/testing-library/react-testing-library/blob/main/LICENSE)

## Overview

React Testing Library (RTL) is a thin layer over `react-dom` that renders components into a real (or jsdom) DOM and queries them the way a user would: by role, label, and visible text — never by component internals[^1]. Kent C. Dodds released it in March 2018 as an explicit reaction to Enzyme, whose `shallow()` rendering and instance/state assertions coupled tests to implementation details and broke on every refactor. RTL's guiding principle — "the more your tests resemble the way your software is used, the more confidence they can give you"[^2] — won that argument decisively: Enzyme never shipped an official React 17+ adapter and is effectively dead[^3], while RTL ships in the default templates of essentially every React scaffold.

The library itself is deliberately small. Almost all query logic lives in `@testing-library/dom`, which is framework-agnostic and shared by sibling libraries (Vue, Angular, Svelte flavors). RTL contributes only React-specific glue: `render`, `act` wiring, `rerender`, `renderHook`, and cleanup. This smallness is the defining tradeoff: RTL is opinionated by omission. There is no shallow rendering, no way to read component state, no instance access — if you want those, the library's position is that your test is wrong, not the API.

At ~19.6k stars with a last push in April 2026, the repo looks quiet — dozens of commits a year, 81 open issues — but that reflects a feature-complete surface tracking React's release cadence rather than abandonment. Real activity spikes around React majors (18's `createRoot`, 19's `act` semantics), since RTL must chase React's rendering internals each cycle.

## Getting Started

```bash
npm install --save-dev @testing-library/react @testing-library/dom
# recommended companions:
npm install --save-dev @testing-library/jest-dom @testing-library/user-event
```

Since v16, `@testing-library/dom` is a peer dependency you install yourself[^4]. React 18+ requires RTL 13+; older React needs `@testing-library/react@12`.

```jsx
import {render, screen} from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import '@testing-library/jest-dom'
import Counter from '../counter'

test('increments on click', async () => {
  const user = userEvent.setup()
  render(<Counter />)

  await user.click(screen.getByRole('button', {name: /increment/i}))

  expect(screen.getByText(/count: 1/i)).toBeInTheDocument()
})
```

Runs under Jest or Vitest with a jsdom (or happy-dom) environment; RTL itself is runner-agnostic.

## Architecture / How It Works

`render(<Component />)` appends a container `div` to `document.body` and mounts the component with `react-dom`'s `createRoot` (v13+; `ReactDOM.render` before that[^5]), wrapping the mount in React's `act()` so effects and state updates flush before your first assertion. It returns `rerender`, `unmount`, `asFragment`, and bound queries — though the idiomatic style is the global `screen` object, whose queries are pre-bound to `document.body`.

The query system comes entirely from `@testing-library/dom`: a matrix of variants (`getBy*` throws if absent, `queryBy*` returns null, `findBy*` polls asynchronously, plus `*AllBy*` forms) crossed with selectors ordered by intentional priority — `ByRole` first (which computes ARIA accessible names), then `ByLabelText`, `ByText`, and `ByTestId` as the escape hatch. The priority ordering is an accessibility nudge: tests that can't find elements by role usually indicate markup screen readers can't navigate either.

Event handling has two tiers. `fireEvent` dispatches a single raw DOM event wrapped in `act`. `@testing-library/user-event` (a separate package, recommended since its v14 rewrite) simulates full interaction sequences — a `click` produces pointer, focus, mouse, and click events in browser order — catching bugs `fireEvent` misses[^6]. Cleanup (unmounting and container removal) auto-registers via `afterEach` when the test framework exposes it.

`renderHook` was absorbed into core in v13.1 when `react-hooks-testing-library` was retired; it mounts a throwaway component that calls your hook and exposes `result.current`.

The coupling story: RTL depends on React's unstable internals only through `act`, but `act`'s semantics shift with React versions (sync in 17, async-aware in 18, further changed in 19). Each React major forces a matching RTL major, and mismatched pairs produce cryptic failures.

## Production Notes

- **`act` warnings are the perennial footgun.** "An update to X was not wrapped in act(...)" almost always means a state update landed after the test finished — an unawaited promise, timer, or subscription. The fix is `await findBy*` / `waitFor`, not manual `act()` wrapping (which RTL already does) and not silencing the warning.
- **`waitFor` misuse causes flakes.** Side effects inside `waitFor` callbacks re-run on every poll; assertions after `fireEvent` without awaiting async updates pass or fail by timing. `eslint-plugin-testing-library` catches most of these statically and belongs in every RTL codebase.
- **`getByRole` is the slowest query.** Accessible-name computation over large DOM trees is expensive; test suites dominated by `ByRole` on big component trees can run multiples slower than `ByTestId` equivalents. Usually acceptable; occasionally worth profiling with `screen.debug` and narrowing `within()` scopes.
- **jsdom is not a browser.** No layout (`getBoundingClientRect` returns zeros), no real navigation, historically missing observers (`IntersectionObserver`, `ResizeObserver` need polyfills or mocks). CSS-dependent behavior — visibility, media queries, sticky positioning — cannot be meaningfully tested; that work belongs in Playwright or Cypress.
- **Upgrade pains cluster on React majors.** v13 (React 18) changed `render` to `createRoot`, making previously-sync updates async and breaking timing-dependent tests en masse. v16 moved `@testing-library/dom` to a peer dependency — silently duplicated or missing installs caused "element not found" errors from mismatched instances[^4].
- **jest-dom is practically mandatory.** Without `toBeInTheDocument`, `toHaveTextContent`, etc., failure messages degrade to generic equality errors. It is a separate install by design but omitting it is rarely intentional.

## When to Use / When Not

**Use when:**
- Testing React component behavior at the unit/integration level — this is the community default and the assumption behind most documentation and AI-generated test code.
- You want tests to survive refactors: markup-and-behavior assertions don't care whether state is a hook, reducer, or external store.
- You need an accessibility forcing function; `ByRole`-first queries surface unlabeled controls immediately.
- Testing custom hooks via `renderHook` (for reusable hooks — single-use hooks are better tested through their consuming component).

**Avoid when:**
- The behavior depends on real layout, CSS, scrolling, or navigation — jsdom cannot express it; use Playwright or Cypress component/E2E tests.
- You genuinely need to assert on component internals (rare, but e.g. testing a state-machine library's transitions) — RTL structurally forbids it.
- You're on React Native — use `@testing-library/react-native`, a separate codebase.
- Your suite's value is visual — screenshot/DOM-snapshot testing via RTL's `asFragment` is brittle and better served by dedicated visual-regression tooling.

## Alternatives

- enzymejs/enzyme — historical predecessor; use only if maintaining a legacy React ≤17 suite that already depends on it. Unmaintained, no official React 17+ adapter[^3].
- cypress-io/cypress — use instead when component tests must run in a real browser with real CSS and layout.
- microsoft/playwright — use instead for end-to-end coverage, or its component-testing mode when jsdom fidelity is insufficient.
- vitest-dev/vitest — not a replacement but the runner pairing; its browser mode can host RTL tests in real Chromium instead of jsdom.
- storybookjs/storybook — use instead when interaction tests should double as living documentation (play functions use Testing Library queries internally).

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-03 | Initial release by Kent C. Dodds as an Enzyme alternative[^1]. |
| 8.0 | 2019-05 | Moved from `react-testing-library` to the `@testing-library/react` scope under the new testing-library org. |
| 9.0 | 2019 | `jest-dom` matchers fully split out; extend-expect import path changes. |
| 13.0 | 2022 | React 18: `createRoot` by default, dropped React ≤17 (use v12)[^5]. |
| 13.1 | 2022 | `renderHook` added to core; `react-hooks-testing-library` retired. |
| 14.0 | 2023 | Maintenance major; dropped older Node.js versions and deprecated APIs. |
| 15.0 | 2024 | Minimum Node 18; React 19 preparation. |
| 16.0 | 2024-06 | `@testing-library/dom` became a peer dependency — install it explicitly[^4]. |

## References

[^1]: Kent C. Dodds, "Introducing the react-testing-library" — 2018. https://kentcdodds.com/blog/introducing-the-react-testing-library
[^2]: Testing Library, "Guiding Principles". https://testing-library.com/docs/guiding-principles/
[^3]: Wojciech Maj (Enzyme adapter author), "Enzyme is dead. Now what?" — 2022. https://dev.to/wojtekmaj/enzyme-is-dead-now-what-4ehl
[^4]: react-testing-library v16.0.0 release notes. https://github.com/testing-library/react-testing-library/releases/tag/v16.0.0
[^5]: react-testing-library v13.0.0 release notes. https://github.com/testing-library/react-testing-library/releases/tag/v13.0.0
[^6]: Testing Library, "user-event — Differences from fireEvent". https://testing-library.com/docs/user-event/intro/

## Tags

javascript, react, testing, unit-testing, component-testing, jsdom, accessibility, developer-tools, dom-testing
