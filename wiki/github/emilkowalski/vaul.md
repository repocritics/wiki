# emilkowalski/vaul

> An unstyled drawer / bottom-sheet component for React, built on Radix Dialog with drag-to-dismiss gestures.

[GitHub repo](https://github.com/emilkowalski/vaul) ·
[Official website](https://vaul.emilkowal.ski) ·
[License: MIT](https://github.com/emilkowalski/vaul/blob/main/LICENSE)

## Overview

Vaul is a drawer primitive for React by Emil Kowalski, the author of the `sonner`
toast library. It provides the iOS-style bottom sheet — a panel that slides up
from an edge, can be dragged with a pointer/touch, snaps to points, and dismisses
by flinging it past a velocity threshold. It ships almost no visual styling; you
bring your own CSS. Its most visible deployment is the `Drawer` component in
shadcn/ui, which is a thin styled wrapper over vaul[^1].

The defining design choice is that vaul is not a standalone widget — it is built
on top of Radix UI's Dialog primitive[^2]. That buys it focus trapping, portal
rendering, escape-to-close, `aria` wiring, and scroll locking for free, and vaul
layers the drag physics, snap points, and background-scaling effect on top. The
tradeoff is that vaul inherits Radix Dialog's mental model (Root / Trigger /
Portal / Overlay / Content) and its peer-dependency surface.

The important caveat for any 2026 adopter: the maintainer has publicly marked the
project unmaintained — the README calls it a hobby project he does not have time
to work on "in the near future"[^3]. It is widely used and stable, but new
features and most bug fixes are not landing (roughly 157 open issues at this
writing). Treat it as mature-but-frozen.

## Getting Started

```bash
npm install vaul
# peer: react, react-dom (Radix Dialog is bundled as a dependency)
```

```tsx
import { Drawer } from "vaul";

export function MyDrawer() {
  return (
    <Drawer.Root>
      <Drawer.Trigger>Open</Drawer.Trigger>
      <Drawer.Portal>
        <Drawer.Overlay className="fixed inset-0 bg-black/40" />
        <Drawer.Content className="fixed bottom-0 left-0 right-0 rounded-t-xl bg-white p-4">
          <Drawer.Handle />
          <Drawer.Title>Title</Drawer.Title>
          <p>Drag me down to dismiss.</p>
        </Drawer.Content>
      </Drawer.Portal>
    </Drawer.Root>
  );
}
```

The component is positioned entirely by your own CSS (`fixed bottom-0 …`). Vaul
supplies the transform, the drag handling, and the open/close state; layout is
yours.

## Architecture / How It Works

Vaul wraps `@radix-ui/react-dialog`. `Drawer.Root` renders a `Dialog.Root` and
the rest of the parts map onto Radix's Trigger/Portal/Overlay/Content, so the
accessibility and mounting behavior are Radix's, not vaul's own[^2]. The
drawer-specific behavior is added in a controller that listens to pointer events
on the content element:

- **Drag translation.** On pointer-down/move it applies a `transform:
  translate3d(...)` to the content, following the pointer with damping past the
  open position. Release computes velocity; past a threshold it animates closed,
  otherwise it snaps back or to the nearest snap point.
- **Snap points.** `snapPoints` (numbers as viewport fractions or px strings)
  plus `activeSnapPoint` let the sheet rest at intermediate heights. This is the
  half-open / full-open pattern seen in native mobile sheets.
- **Background scaling.** The signature effect — scaling and pushing back the
  page behind the sheet — requires marking a wrapper element with
  `[vaul-drawer-wrapper]` (or `setBackgroundColorOnScale`), because vaul mutates
  that element's `transform` and `border-radius` while the drawer is open.
- **Direction.** `direction` supports `bottom` (default), `top`, `left`, and
  `right`, so the same primitive serves side sheets and top notifications.
- **Scroll coordination.** Inside a scrollable drawer, vaul only starts a drag
  when the inner scroller is at its boundary, so content scroll and sheet drag do
  not fight; elements opt out of drag with `data-vaul-no-drag`.

Body-scroll locking is handled to prevent the page behind from scrolling on
touch; historically this is the most fragile area on iOS Safari, where the
platform's rubber-band scrolling and the on-screen keyboard fight the lock.

## Production Notes

- **Unmaintained.** The maintainer has flagged the repo as not actively worked
  on[^3]. Pin a version you have tested; do not assume regressions will be fixed
  upstream. If you need guaranteed maintenance, budget for a fork.
- **Background scaling is a footgun.** The scale effect silently does nothing
  unless a parent has `vaul-drawer-wrapper` set *and* a solid background color —
  a transparent or unmarked wrapper produces no visible effect and no warning.
- **iOS Safari keyboard/input handling.** Focusing an input inside a bottom sheet
  can cause the viewport to jump or the sheet to be pushed by the software
  keyboard. This is the most-reported class of issue; test real iOS devices, not
  just desktop responsive mode.
- **Nested and stacked drawers.** Nested drawers and drawer-over-dialog stacking
  work but are finicky; overlay z-index and scroll-lock ownership between the
  Radix layers need manual attention.
- **Scroll-lock side effects.** Because vaul locks body scroll, it can interact
  badly with other libraries that also manipulate `overflow`/`position` on
  `body`. Layout shift from scrollbar compensation is possible on desktop.
- **Peer surface.** You inherit Radix Dialog's version constraints and React
  version support; on React 19, verify your pinned vaul version behaves.

## When to Use / When Not

**Use when:**
- You want a native-feeling, drag-dismissible bottom sheet on the web without
  writing gesture physics yourself.
- You already use (or are fine adopting) Radix Dialog semantics.
- You use shadcn/ui and want its `Drawer` — you are already using vaul.

**Avoid when:**
- You need a component with an active maintenance guarantee and SLA on fixes.
- You only need a plain modal or side panel with no drag gesture — use Radix
  Dialog directly and skip the extra layer.
- You are on React Native — vaul is DOM/pointer-event based and does not apply.

## Alternatives

- radix-ui/primitives — the Dialog primitive vaul is built on; use it directly
  when you want an accessible modal/side panel and do not need drag-to-dismiss.
- shadcn-ui/ui — its `Drawer` is a styled wrapper over vaul; use when you want
  vaul with opinionated styling copy-pasted into your codebase rather than a dep.
- dotcore64/react-spring-bottom-sheet — react-spring-based bottom sheet; use when
  you want an actively-alternative sheet with spring config control.
- framer/motion — build a bespoke sheet with full gesture/animation control when
  vaul's conventions do not fit and you can afford the implementation cost.
- mui/material-ui — its `SwipeableDrawer` is a maintained, Material-styled option
  when you are already in the MUI ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-07 | Repository created; bottom drawer on Radix Dialog[^4]. |
| 0.x | 2023–2024 | Snap points, directions (top/left/right), nested drawers, background scaling matured. |
| 1.0 | 2024 | First 1.0 release line[^5]. |
| — | 2025 | README updated to mark the project unmaintained[^3]. |

## References

[^1]: shadcn/ui Drawer component (wraps vaul). https://ui.shadcn.com/docs/components/drawer
[^2]: Radix UI Dialog primitive. https://www.radix-ui.com/primitives/docs/components/dialog
[^3]: vaul README — "This repo is unmaintained" note. https://github.com/emilkowalski/vaul
[^4]: emilkowalski/vaul repository (created 2023-07-16). https://github.com/emilkowalski/vaul
[^5]: vaul releases. https://github.com/emilkowalski/vaul/releases

## Tags

react, typescript, drawer, bottom-sheet, dialog, ui-component, radix, mobile-ui, gestures, unmaintained
