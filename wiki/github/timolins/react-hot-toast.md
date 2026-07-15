# timolins/react-hot-toast

> Imperative toast notifications for React — call `toast()` from anywhere, render one `<Toaster />`, styled and accessible by default.

[GitHub repo](https://github.com/timolins/react-hot-toast) ·
[Official website](https://react-hot-toast.com) ·
[License: MIT](https://github.com/timolins/react-hot-toast/blob/main/LICENSE)

## Overview

react-hot-toast is a small notification library for React by Timo Lins, first
released in December 2020[^1]. Its core idea is a decoupled imperative API: you
mount a single `<Toaster />` somewhere in the tree, then fire notifications from
anywhere — event handlers, async functions, non-component modules — by importing
and calling `toast()`. There is no context provider to thread through and no ref
to pass around; the emitter and the renderer communicate through a module-level
store.

The library is deliberately narrow. It ships styled, animated toasts that look
finished out of the box (roughly 5 kB minzipped including styles[^2]), a
`toast.promise()` helper that swaps loading → success/error states from a
promise, and a set of headless hooks (`useToaster`, `useToasterStore`) for teams
that want the state machine but not the default UI. The defining tradeoff is
scope versus control: it is one of the fastest paths to good-looking toasts, but
it is React-DOM only, has a small surface for advanced layout, and its styling
is applied through a CSS-in-JS runtime (goober) rather than plain classes, which
complicates deep visual overrides.

As of 2026 the project has roughly 11k stars and 372 forks. Release cadence has
slowed — the gap between 2.4.1 (2023) and 2.5.1 (2024) was over eighteen months
— but it remains maintained, with 2.6.0 shipping in August 2025[^3].

## Getting Started

```sh
npm install react-hot-toast
# or: pnpm add react-hot-toast
```

Mount one `<Toaster />`, then call `toast()` from anywhere:

```jsx
import toast, { Toaster } from 'react-hot-toast';

function App() {
  return (
    <div>
      <button onClick={() => toast('Saved.')}>Save</button>
      <button
        onClick={() =>
          toast.promise(saveSettings(), {
            loading: 'Saving…',
            success: 'Settings saved',
            error: 'Could not save',
          })
        }
      >
        Save async
      </button>
      <Toaster position="top-center" />
    </div>
  );
}
```

`toast.success()`, `toast.error()`, `toast.loading()`, and `toast.custom()`
cover the common cases; `toast.dismiss(id)` and `toast.remove(id)` retire them.
Passing a stable `id` updates an existing toast instead of stacking a new one.

## Architecture / How It Works

The heart of the library is a single module-level store driven by a reducer. The
`toast()` functions do not render anything — they dispatch actions (`ADD_TOAST`,
`UPDATE_TOAST`, `DISMISS_TOAST`, `REMOVE_TOAST`) to that store. `<Toaster />`
subscribes to the store via `useToaster()` and renders the current list. Because
the store lives at module scope rather than in React context, emitters can be
plain functions in any file, and there is exactly one source of truth per bundle.

Layout is height-aware. Each rendered toast reports its measured height back to
the store, and the `Toaster` computes vertical offsets so entries stack and
reflow smoothly as they mount, resize, or leave. Enter/exit animations are CSS
keyframes; the exit is delayed so a dismissed toast can animate out before it is
removed from the store.

Styling goes through **goober**, a ~1 kB CSS-in-JS runtime, which is why the
default toasts carry their own look with no stylesheet import. You override
appearance through `toastOptions` on the `Toaster` (per-type `style`/`className`),
per-call options, or by dropping to `toast.custom()` and rendering your own JSX.
The headless path — `useToaster()` / `useToasterStore()` — exposes the toast
list plus the hover/focus handlers that pause auto-dismiss, letting you build an
entirely custom renderer while reusing the timing and accessibility logic.

Accessibility is handled by announcing toast content through an ARIA live region
and pausing dismissal timers on pointer/focus interaction, so screen-reader users
are not raced by the auto-dismiss clock.

## Production Notes

- **Render exactly one `<Toaster />`.** The store is global, so a second Toaster
  renders the same toasts twice. Mount it once near the app root (in a client
  component under Next.js App Router).
- **Not a portal by default.** The Toaster renders a `position: fixed` container
  inline where you mount it. A transformed or `filter`-ed ancestor creates a new
  containing block/stacking context that can trap or clip it; if toasts appear in
  the wrong place or behind other UI, mount the Toaster high in the tree and
  check ancestor `transform`/`z-index`. The container z-index is high but finite.
- **Default durations differ by type.** Blank and error toasts auto-dismiss after
  a few seconds, success sooner, and `toast.loading()` persists until you resolve
  or dismiss it. `toast.promise()` manages this for you; hand-rolled loading
  toasts must be dismissed explicitly or they stay forever.
- **Deep style overrides fight the runtime.** Because styles come from goober,
  targeting internal structure with your own CSS can be brittle. Prefer
  `toastOptions`, per-call `style`, or `toast.custom()` for anything beyond color
  and spacing tweaks.
- **React DOM only.** There is no official React Native support and no Vue/Svelte
  port; the package assumes a DOM renderer. It also expects a browser environment
  at call time — `toast()` invoked during SSR has nothing to emit into.
- **Duplicate-suppression is manual.** Rapid identical events stack unless you
  pass a shared `id` so repeats update one toast rather than piling up.

## When to Use / When Not

**Use when:**
- You want good-looking toasts quickly with almost no configuration.
- You need to fire notifications from outside the component tree (services,
  interceptors, stores) without wiring context.
- You want a built-in promise → loading/success/error flow.
- You want the option to go headless later without changing the emit API.

**Avoid when:**
- You need React Native or a non-React renderer.
- You require heavy per-toast layout control, queuing policies, or progress bars
  out of the box — a more feature-dense library fits better.
- Your design system mandates plain-class styling and forbids a CSS-in-JS runtime.
- You want the newer stacked/expandable toast interaction that Sonner popularized.

## Alternatives

- emilkowalski/sonner — newer, opinionated stacked/expandable toasts; the common
  default in shadcn/ui projects. Use when you want that interaction model.
- fkhadra/react-toastify — older and more feature-dense (progress bars, more
  container options). Use when you need configurability over minimal footprint.
- radix-ui/primitives — unstyled, accessible Toast primitive. Use when you want
  to own all styling and behavior and only need the accessibility scaffolding.
- shadcn-ui/ui — its `useToast`/Toast component (Radix-based). Use when you are
  already committed to the shadcn component set.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.5.0 | 2020-12 | Early pre-1.0 releases. |
| 1.0.0 | 2020-12-18 | First stable release[^1]. |
| 2.0.0 | 2021-05-31 | Major release; headless `useToaster()` and expanded API. |
| 2.1.0 | 2021-07-31 | Follow-up feature release. |
| 2.2.0 | 2022-01-04 | Feature/maintenance release. |
| 2.3.0 | 2022-07-12 | Feature release. |
| 2.4.0 | 2022-09-03 | Feature release. |
| 2.4.1 | 2023-04-28 | Patch. |
| 2.5.1 | 2024-12-31 | Resumed releases after a long gap. |
| 2.5.2 | 2025-02-15 | Patch. |
| 2.6.0 | 2025-08-15 | Latest release as of this writing[^3]. |

## References

[^1]: react-hot-toast v1.0.0 release — 2020-12-18. https://github.com/timolins/react-hot-toast/releases/tag/v1.0.0
[^2]: react-hot-toast documentation, "less than 5kb including styles." https://react-hot-toast.com/
[^3]: react-hot-toast v2.6.0 release — 2025-08-15. https://github.com/timolins/react-hot-toast/releases/tag/v2.6.0

## Tags

react, typescript, toast, notifications, snackbar, frontend, ui, react-dom, css-in-js, headless-hooks, promise-api
