# pmndrs/leva

> A React hook–driven GUI panel for tweaking values at runtime — the debug-knob standard of the react-three-fiber ecosystem.

[GitHub repo](https://github.com/pmndrs/leva) ·
[Official website](https://leva.pmnd.rs) ·
[License: MIT](https://github.com/pmndrs/leva/blob/main/LICENSE)

## Overview

Leva is a controls-panel library for React from Poimandres (pmndrs), the collective behind react-three-fiber, zustand, and drei[^1]. You declare values with the `useControls` hook and Leva renders a floating panel of inputs — sliders, color pickers, vector fields, dropdowns — wired back to those values. It is the tool most r3f demos reach for when they need to expose "tweak this number and watch the scene react" without hand-building a form.

The defining characteristic is *inference*: Leva looks at the shape of a value and picks an input. A number becomes a slider, `'#ff0000'` becomes a color picker, `{ x: 0, y: 0 }` becomes a 2D vector pad, an array or `{ options }` becomes a select. This makes the happy path extremely short — one line of config, no wiring — at the cost of a config schema that is implicit and occasionally surprising when inference guesses wrong.

Leva is a *developer* tool, not an end-user form library. It has no validation story, no submit lifecycle, no accessibility contract beyond keyboard navigation, and it has never shipped a 1.0 — the README still carries an "under heavy development" banner years on[^2], and the version has sat in the 0.9.x/0.10.x range. In practice it is stable and widely used, but the pre-1.0 status is an honest signal that its API and internals are not frozen.

## Getting Started

```bash
npm i leva
```

```jsx
import { useControls } from 'leva'

function Scene() {
  // Values are inferred: number -> slider, string '#rrggbb' -> color,
  // {x,y} -> vector, {options} -> select.
  const { speed, color, position } = useControls({
    speed: { value: 1, min: 0, max: 10, step: 0.1 },
    color: '#ff0055',
    position: { x: 0, y: 0 },
    quality: { options: ['low', 'high'] },
  })

  return <mesh position={[position.x, position.y, 0]} />
}
```

The panel renders itself automatically the first time `useControls` runs; no provider or `<Leva />` mount is required for the default single panel. Mount `<Leva />` explicitly only when you want to configure or reposition it.

## Architecture / How It Works

Leva is a pnpm monorepo. The core packages are `leva` (React bindings and the default input set), `@leva-ui/plugin-*` add-ons (plot, spring, dates, bezier), and an internal `@leva-ui/plugin-utils` for authoring inputs[^3]. Styling is done with stitches (`@stitches/react`), a near-zero-runtime CSS-in-JS library, which is why the theme is a single JS object passed to `<Leva theme={...} />` rather than a stylesheet.

State lives in a Leva store, not React state. `useControls` writes your schema into that store and subscribes the component to the fields it reads. Two consequences follow:

- **Transient updates.** Supplying an `onChange` handler on a field makes that field *transient*: the store calls your handler on every change but does **not** trigger a re-render of the host component. This is the intended pattern for animation loops and r3f, where re-rendering React 60 times a second would be a mistake. The returned value is only meaningful for the non-transient case.
- **Imperative setters.** Passing a function as the second `useControls` argument returns a `[values, set]` tuple, letting you push values into the panel from outside (e.g. syncing a slider to a gamepad).

Because the store is separate, you can run multiple independent panels with `useCreateStore` + `<LevaPanel store={...} />`, and you can hoist controls out of the component tree entirely. Helpers like `folder()`, `button()`, `buttonGroup()`, and `monitor()` are schema-level constructs, not components — they exist to structure the panel and to read values Leva does not own.

The inference engine is the conceptual heart and the main footgun. The mapping from value shape to input type is a fixed set of "special input" detectors; when your data looks like something Leva recognizes but isn't (a plain `{x, y}` object you meant to keep opaque, a hex-like string), it will render the wrong widget and you must disambiguate with an explicit `{ value, ... }` descriptor.

## Production Notes

Leva is a build-time / debug dependency for most teams, and that framing matters more than any single caveat:

- **Ship it out, or ship it hidden.** There is no tree-shaking magic that removes Leva from a production bundle if you still import `useControls`. Common patterns are to gate the panel behind a dev flag, render `<Leva hidden />`, or strip the controls before release. Leaving a full panel in a shipped consumer app is almost always unintended.
- **React 18 `createRoot` warning.** Leva self-mounts its panel, which historically produced a console error under React 18's strict root handling. It is cosmetic and safe to ignore, or resolvable per the maintainers' issue thread[^4]. Still a frequent first-run surprise.
- **Re-render discipline.** The value returned by `useControls` re-renders the host on every change. In hot paths (per-frame r3f code) you almost always want the `onChange` transient form instead; getting this wrong shows up as jank, not an error.
- **Styling is stitches, not CSS.** Theming goes through the `theme` prop and stitches tokens. There is no documented plain-CSS override path, so deep visual customization means learning the theme object; class-name targeting is brittle across versions.
- **Pre-1.0 churn.** Because the project never declared 1.0, minor bumps have carried behavior changes without semver-major signaling. Pin the version and read the changelog before upgrading; the last published release predates a long quiet period — pushes to the repo have slowed markedly, so treat it as mature-but-lightly-maintained rather than actively evolving[^5].

## When to Use / When Not

**Use when:**
- You are building a react-three-fiber / React canvas demo and want live knobs with near-zero setup.
- You need a debug panel to expose parameters to designers or yourself during development.
- The values you tweak map cleanly onto Leva's inferred input types (numbers, colors, vectors, selects).

**Avoid when:**
- You need a real user-facing form with validation, submission, and accessibility guarantees — reach for a form library instead.
- You are not using React; Leva is React-only by design.
- You want a stable, frozen API with an official 1.0 and active maintenance cadence.
- You need deep visual customization outside the stitches theme system.

## Alternatives

- cocopon/tweakpane — vanilla-JS panel with a richer built-in input set (monitors, graphs); use when you are not on React or want more polished sliders out of the box.
- georgealways/lil-gui — modern, tiny, framework-agnostic successor to dat.gui; use for plain JS/three.js debug UIs.
- dataarts/dat.gui — the original three.js control panel, now effectively legacy; use only for parity with old examples.
- react-hook-form/react-hook-form — use instead when the panel is actually a user-facing form that needs validation and submit handling, not a debug tool.
- pmndrs/zustand — not a GUI, but if you only need external, subscribable state without a panel, drop the UI layer entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-11-07 | Initial pmndrs repository[^6]. |
| 0.9.x | 2021 | Public `useControls` API, plugin packages, stitches theming. |
| 0.10.x | 2023 | Continued 0.x line; still flagged "heavy development", no 1.0. |
| — | 2025-11 | Last repository push; development pace slowed[^5]. |

## References

[^1]: Poimandres (pmndrs) open-source collective. https://pmnd.rs
[^2]: Leva README, "This repo is under heavy development" banner. https://github.com/pmndrs/leva#readme
[^3]: Leva packages and plugins on npm (`leva`, `@leva-ui/plugin-plot`, `-spring`, `-dates`, `-bezier`). https://www.npmjs.com/package/leva
[^4]: Leva issue #358 — React 18 `createRoot` console error. https://github.com/pmndrs/leva/issues/358
[^5]: Repository metadata (last push 2025-11-09), GitHub API `repos/pmndrs/leva`. https://github.com/pmndrs/leva
[^6]: Repository creation date 2020-11-07, GitHub API. https://github.com/pmndrs/leva

## Tags

react, gui, controls-panel, typescript, react-three-fiber, debug-tools, hooks, stitches, dat-gui-alternative, frontend
