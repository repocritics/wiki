# fkhadra/react-toastify

> An imperative toast/notification library for React: call `toast("...")` from anywhere, render one `<ToastContainer />`.

[GitHub repo](https://github.com/fkhadra/react-toastify) ·
[Official website](https://fkhadra.github.io/react-toastify/introduction) ·
[License: MIT](https://github.com/fkhadra/react-toastify/blob/main/LICENSE)

## Overview

react-toastify is one of the oldest and most widely installed toast/notification
libraries for React, created by Fadi Khadra in 2016[^1]. Its defining design
choice is an *imperative* API decoupled from the component tree: you mount a
single `<ToastContainer />` once, then trigger notifications by calling the
`toast()` function from anywhere — event handlers, async callbacks, service
modules, even code outside React. This is convenient precisely because it side-
steps React's normal data flow, and that is also its central tension: toasts are
global mutable UI state driven by an event emitter rather than props or context.

The library targets the common case — success/error/info banners, form feedback,
async operation status — and optimizes for getting something on screen in a few
lines. It bundles its own CSS (with `Toastify__*` class names and CSS variables),
built-in transitions, a progress bar tied to auto-dismiss timing, dark mode,
RTL, swipe-to-close, and stacked notifications. It is written in TypeScript and
ships types. As of 2026 it sits around 13.4k stars with steady maintenance, and
competes in a category that has grown more crowded and more opinionated since it
first shipped.

## Getting Started

```bash
npm install react-toastify
# or: yarn add react-toastify / pnpm add react-toastify
```

```jsx
import { ToastContainer, toast } from "react-toastify";
import "react-toastify/dist/ReactToastify.css"; // REQUIRED — see Production Notes

function App() {
  const notify = () => toast.success("Saved!");
  return (
    <div>
      <button onClick={notify}>Notify</button>
      <ToastContainer position="top-right" autoClose={3000} />
    </div>
  );
}
```

The stylesheet import is not optional. Without it, toasts render as unstyled,
mispositioned DOM and appear "broken" — this is the single most common support
question. The exact import path has changed between major versions; check the
docs for the version you installed[^2].

## Architecture / How It Works

The core is an **event-emitter store**, not React state or context. `toast()` and
its variants (`toast.success`, `toast.error`, `toast.info`, `toast.warning`,
`toast.loading`, `toast.promise`) dispatch events to an internal manager.
`<ToastContainer />` subscribes to that manager, holds the active toast list in
its own state, owns timers and DOM refs, and renders each toast into a fixed-
position wrapper. Because the emitter is a module-level singleton, `toast()`
works before or after the container renders and from code with no access to the
React tree — the property most teams adopt the library for.

Each `toast()` call returns an **id**. That id is the handle for the imperative
lifecycle: `toast.update(id, {...})` mutates an existing toast in place (common
with `toast.promise` to swap a spinner for a result), `toast.dismiss(id)` removes
one, `toast.isActive(id)` checks presence. Passing your own `toastId` makes calls
idempotent, which is the standard way to prevent duplicates.

Other internals worth knowing:

- **Positioning** — the container is `position: fixed`; `position` prop picks one
  of six anchors. Multiple independent containers are supported by giving each a
  `containerId` and routing a toast with `{ containerId }`.
- **Progress bar & timers** — auto-dismiss is a CSS animation whose duration is
  `autoClose`; hovering (or window blur, with `pauseOnFocusLoss`) toggles
  `animation-play-state` to pause it. The bar can be driven manually for
  nprogress-style control.
- **Transitions** — a `cssTransition` helper builds enter/exit animations from
  class names; you can swap in your own (e.g. animate.css) or the bundled Bounce/
  Slide/Zoom/Flip.
- **Theming** — styling is CSS variables (`--toastify-color-*`) plus overridable
  `Toastify__*` classes; `theme="dark"` and per-toast `className`/`style` are the
  escape hatches. There is no headless/render-prop mode — you theme the shipped
  markup rather than supplying your own.
- **`limit`** — caps simultaneous toasts and queues the overflow.

## Production Notes

- **Forgetting the CSS import** is the number-one footgun. It fails silently into
  unstyled, wrongly-positioned toasts rather than an error.
- **SSR / Next.js App Router.** `ToastContainer` is client-only; the file must be
  a Client Component (`"use client"`). Rendering it in a Server Component, or
  calling `toast()` during render, produces hydration mismatches or lost toasts.
  Mount one container in a client layout and trigger from client handlers.
- **StrictMode double-invocation.** In React 18 dev mode, effects run twice; a
  `toast()` inside `useEffect` without a stable `toastId` shows two toasts. Dedupe
  with an explicit `toastId` or move the call out of an effect.
- **One container, not many.** Accidentally mounting `<ToastContainer />` in
  several components leads to duplicate or missing toasts. Use a single container
  at the app root, or `containerId` when you deliberately want more than one.
- **Upgrade friction across majors.** v9 was a rewrite that changed props and
  animation APIs; later majors adjusted the CSS import path and theming approach.
  Pin the version and read the changelog before bumping — visual regressions are
  easy to miss[^3].
- **z-index & modals.** The fixed container can render under (or over) dialog
  overlays depending on stacking context; the `--toastify-z-index` variable is
  the intended override.
- **Accessibility.** Toasts set ARIA roles, but auto-dismissing, low-contrast, or
  purely-visual notifications are inherently hard for screen-reader and motion-
  sensitive users. Give important messages long/no `autoClose` and text content.

## When to Use / When Not

**Use when:**
- You want to fire notifications imperatively from anywhere, including non-React
  code, with almost no wiring.
- The bundled look-and-feel (or light CSS-variable theming) is good enough.
- You need built-ins like `toast.promise`, update-in-place, stacking, swipe-to-
  close, and a progress bar without building them.

**Avoid when:**
- You want fully headless, bring-your-own-markup toasts governed by React state —
  reach for a primitive instead.
- You are minimizing bundle size and want the smallest possible dependency.
- Accessibility is a hard requirement and you need full control over roles, focus,
  and announcement timing.

## Alternatives

- timolins/react-hot-toast — smaller, promise-friendly API, minimal styling; the
  usual pick when you want lighter weight than react-toastify.
- emilkowalski/sonner — modern, opinionated, stacked-by-default; use it when you
  want the shadcn/ui-style look with little configuration.
- radix-ui/primitives — the headless, accessibility-first Toast primitive; use it
  when you must own the markup, styling, and ARIA behavior.
- iamhosseindhv/notistack — snackbar-style stacking built around Material UI; use
  it when you are already in an MUI codebase.
- jossmac/react-toast-notifications — older provider/hook API; largely superseded,
  relevant mostly for legacy migrations.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Initial release; imperative `toast()` + `ToastContainer`[^1]. |
| 9.0 | 2022 | Major rewrite; changed props, animations, and theming[^3]. |
| 10.0 | 2024 | Stacked notifications, swipe-to-close, added features[^3]. |
| 11.0 | 2024–25 | Reworked styling/CSS import and theming defaults[^3]. |

Exact release dates and per-version breaking changes are on the GitHub releases
page[^3]; versions above are dated conservatively where I am confident.

## References

[^1]: react-toastify repository and author (Fadi Khadra), created 2016-11-08. https://github.com/fkhadra/react-toastify
[^2]: react-toastify documentation — Introduction / installation, including the required stylesheet import. https://fkhadra.github.io/react-toastify/introduction
[^3]: react-toastify GitHub releases (per-version changelog and dates). https://github.com/fkhadra/react-toastify/releases

## Tags

react, notifications, toast, snackbar, typescript, frontend, ui-component, javascript, npm-package
