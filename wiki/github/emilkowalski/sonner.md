# emilkowalski/sonner

> An opinionated, imperative toast component for React — call `toast()` from anywhere, render one `<Toaster />`.

[GitHub repo](https://github.com/emilkowalski/sonner) ·
[Official website](https://sonner.emilkowal.ski) ·
[License: MIT](https://github.com/emilkowalski/sonner/blob/main/LICENSE)

## Overview

Sonner is a single-purpose React toast (transient notification) library by Emil
Kowalski, first published in early 2023[^1]. Its thesis is that a toast should be
fired imperatively from anywhere in the app — an event handler, a data-fetch
callback, a store subscription — without threading a React context or a hook
through the call site. You mount one `<Toaster />` near the root and call the
module-level `toast()` function from any file. That decoupling is the whole
design.

The "opinionated" label is deliberate: the stacking behavior, the expand-on-hover
interaction, the swipe-to-dismiss gesture, and the default look are chosen for
you and are not trivially reconfigured. This is the library's defining tension.
It looks and feels finished out of the box, and for the common case that is
exactly what you want; but the moment your design system diverges from Emil's
defaults, you are overriding data-attribute-driven CSS rather than composing
primitives. Sonner is not a headless toolkit — it is a finished component with
escape hatches.

Its reach expanded sharply when shadcn/ui deprecated its own Radix-based toast
in favor of recommending Sonner as the default[^2], which is why Sonner now
appears in a large share of new Next.js/React projects without those teams having
chosen it directly. Sonner is React-only; the `vue-sonner` and `svelte-sonner`
packages are separate community ports, not maintained in this repository.

## Getting Started

```bash
npm install sonner
```

```jsx
import { Toaster, toast } from 'sonner';

function App() {
  return (
    <div>
      <Toaster richColors position="top-right" />
      <button onClick={() => toast.success('Saved')}>Save</button>
      <button
        onClick={() =>
          toast.promise(saveToServer(), {
            loading: 'Saving…',
            success: 'Saved',
            error: (err) => `Failed: ${err.message}`,
          })
        }
      >
        Save (async)
      </button>
    </div>
  );
}
```

`toast()` also exposes `toast.error`, `toast.warning`, `toast.info`,
`toast.loading`, `toast.custom(jsx)`, and `toast.dismiss(id)`. Each call returns
an id you can use to update or dismiss that specific toast later.

## Architecture / How It Works

The core is a singleton observer. The `toast()` function does not touch React at
all — it pushes an action onto a module-level store (an `Observer` instance).
`<Toaster />` subscribes to that store on mount and renders the current list of
toasts into an ordered container. Because the store lives at module scope, any
import of `toast` anywhere in the bundle talks to the same instance, which is
what makes the "call from anywhere" ergonomics work without context.

Rendering is a list of absolutely-positioned toast elements whose transforms are
computed from their index and measured heights. The stacked/expanded states, the
enter/exit transitions, and the swipe gesture are driven by CSS custom
properties and `data-*` attributes (`data-sonner-toast`, `data-styled`,
`data-mounted`, `data-front`, etc.) that the component sets, with styles shipped
in an injected stylesheet. Theming and variants (`richColors`, `closeButton`,
`invert`, `theme="dark"`) toggle attributes rather than swapping components.

Accessibility uses an ARIA live region so screen readers announce new toasts;
the keyboard shortcut to focus the toast region is configurable via `hotkey`.
The whole thing is a small dependency-light package (its runtime footprint is
dominated by the component itself, not a tree of sub-dependencies), which is part
of why it slots into projects cleanly.

The consequence of the singleton model: state is global. That is a feature for
call-site ergonomics and a constraint for isolation — see below.

## Production Notes

- **`<Toaster />` must be a Client Component.** In Next.js App Router / RSC it
  needs `"use client"` (either the file it lives in or an ancestor). Mounting it
  in a Server Component throws. The common pattern is to render exactly one
  `<Toaster />` in the root layout's client boundary.
- **Mount exactly one Toaster.** Because the store is a module singleton, two
  mounted `<Toaster />` instances both subscribe and will each render every
  toast — duplicated notifications. This bites people who add a Toaster in a
  nested layout as well as the root.
- **The global store leaks across tests and micro-frontends.** Since `toast()`
  writes to module scope, separate React roots on one page (or tests that don't
  reset the store between cases) share toast state. There is no first-class
  per-instance/scoped store; plan for `toast.dismiss()` in test teardown.
- **Styling is override-by-attribute, not composition.** Custom looks come from
  the `unstyled` prop plus `classNames`/`toastOptions`, or from targeting the
  `data-*` selectors in your own CSS. Deep visual divergence from the defaults
  is more work than it first appears, and default-vs-`unstyled` behavior changed
  across major versions — verify against the version you install.
- **z-index and portals.** Toasts render into a fixed container; interactions
  with modal/dialog stacking contexts and very high `z-index` UI occasionally
  require manual `z-index` tuning.
- **Don't call `toast()` during render.** It mutates external state; call it in
  effects or event handlers. Calling it in a render body causes update-during-
  render warnings and unpredictable ordering.
- **Version churn.** Sonner reached a 2.x line[^3]; prop and default-styling
  changes between the 1.x and 2.x lines mean copied snippets from older tutorials
  may not match current behavior. Pin and read the changelog on upgrade.

## When to Use / When Not

**Use when:**
- You want a finished, good-looking toast with near-zero configuration.
- You like the imperative `toast()`-from-anywhere ergonomics over context/hooks.
- You're already in the shadcn/ui world where Sonner is the recommended default.
- Promise toasts, stacking, and swipe-to-dismiss out of the box are what you need.

**Avoid when:**
- You need a truly headless, fully-composable primitive — reach for a Radix-style
  unstyled toast instead.
- You need multiple isolated toast regions with independent state on one page.
- Your design system diverges hard from the defaults and you'd fight the CSS.
- You're not on React (use a native port or a framework-specific library).

## Alternatives

- timolins/react-hot-toast — closest in spirit: tiny, imperative `toast()` API. Use it when you want an even smaller footprint and don't need Sonner's stacking/expand animations.
- fkhadra/react-toastify — the long-standing feature-heavy classic. Use it when you need extensive per-container configuration and highly customized behavior.
- radix-ui/primitives — the unstyled, accessible Toast primitive. Use it when you want to build the entire toast UI yourself with full control over markup and styling.
- iamhosseindhv/notistack — snackbar stacking built for Material UI. Use it when your app is already on MUI and you want matching snackbars.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-02 | First public release; repository created[^1]. |
| 1.x | 2023–2024 | Stable API: `toast()` singleton, promise toasts, rich colors, positions. |
| — | 2024 | Adopted as the recommended toast in shadcn/ui[^2]. |
| 2.x | 2025 | Major line with prop/default-styling changes[^3]. |

Exact per-release dates are not restated here where they could not be verified
against the source; consult the repository's GitHub Releases for precise tags.

## References

[^1]: Repository metadata, created 2023-02-23. https://github.com/emilkowalski/sonner
[^2]: shadcn/ui — Sonner is documented as the recommended toast component. https://ui.shadcn.com/docs/components/sonner
[^3]: Sonner releases / changelog. https://github.com/emilkowalski/sonner/releases

## Tags

react, toast, notifications, ui-component, typescript, frontend, accessibility, shadcn, client-side, npm-package
