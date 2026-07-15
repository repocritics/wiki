# JedWatson/react-select

> The de facto customizable Select/combobox component for React — flexible, styled, and heavy.

[GitHub repo](https://github.com/JedWatson/react-select) ·
[Official website](https://react-select.com/) ·
[License: MIT](https://github.com/JedWatson/react-select/blob/master/LICENSE)

## Overview

react-select is a controlled Select/dropdown/combobox component for React, created by Jed Watson. It was originally built for KeystoneJS (a Node CMS) and later funded and co-maintained by Thinkmill and Atlassian[^1]. For most of the React ecosystem's history it has been the default answer to "I need a searchable dropdown with multi-select, async loading, tagging, and grouping" — the feature set the native `<select>` cannot provide.

The defining tension is **feature completeness vs. weight and control**. react-select ships single/multi select, search filtering, async option loading, creatable (tag-input) mode, option groups, portalling, and animation out of the box, and exposes a deep customization surface (a styles API, a component-injection API, and controllable state props). The cost is a large runtime: it bundles a CSS-in-JS engine (emotion), renders a non-trivial component tree per option, and has no built-in virtualization — so large option lists and strict bundle budgets are where teams hit friction. It is a "batteries-included widget," not a headless primitive.

As of 2026 the project is stable but in a slow-maintenance phase. Jed Watson stepped back from day-to-day work years ago; Nathan Bierema (`Methuselah96`) has been the primary maintainer. Releases are infrequent and mostly bugfix/typing-focused; the last significant architectural change was the v5 TypeScript rewrite[^4].

## Getting Started

```bash
npm install react-select
# or: yarn add react-select
```

```tsx
import { useState } from 'react';
import Select from 'react-select';

type Option = { value: string; label: string };

const options: Option[] = [
  { value: 'chocolate', label: 'Chocolate' },
  { value: 'strawberry', label: 'Strawberry' },
  { value: 'vanilla', label: 'Vanilla' },
];

export default function FlavorPicker() {
  const [selected, setSelected] = useState<Option | null>(null);
  return (
    <Select
      options={options}
      value={selected}
      onChange={setSelected}
      isSearchable
      isClearable
    />
  );
}
```

Multi-select, async, and creatable modes live in subpath entry points:

```tsx
import AsyncSelect from 'react-select/async';
import CreatableSelect from 'react-select/creatable';
import makeAnimated from 'react-select/animated';
```

## Architecture / How It Works

react-select is a **controlled component with an internal state manager**. Every user-facing prop (`value`, `menuIsOpen`, `inputValue`) has a controllable/uncontrolled pair (`defaultValue`, `defaultMenuIsOpen`, `defaultInputValue`); if you supply the controlled prop you own the state, otherwise the component manages it internally.

Three customization layers, in increasing power:

1. **Styles API** — a `styles` prop with per-slot functions (`control`, `menu`, `option`, `singleValue`, …) that receive base styles and component state and return an emotion style object. `classNamePrefix` alternatively emits stable class names for plain-CSS styling.
2. **`classNames` prop** — added in later v5 to attach class names per slot without going through emotion, easing Tailwind/vanilla-CSS use.
3. **Component Injection API** — a `components` prop lets you replace any internal subcomponent (`Option`, `Control`, `Menu`, `MenuList`, `DropdownIndicator`, `MultiValue`, …) with your own, while inheriting the wiring. This is how virtualization, custom rows, and unusual layouts are built.

Styling runs through **emotion** (`@emotion/react`)[^3] — CSS-in-JS injected at runtime. This is the source of most SSR and CSP complications (see Production Notes). Filtering is done in-memory by default via a `filterOption` function over the `options` array; async mode instead calls a `loadOptions(inputValue)` callback and manages loading state.

v5 is a full **TypeScript rewrite**[^4]. The public component is generic over three type parameters — `Select<Option, IsMulti, Group>` — so the type of `value`/`onChange` narrows based on whether `isMulti` is set. This gives strong inference but produces dense, sometimes hard-to-satisfy generic signatures when you wrap the component.

## Production Notes

**Large option lists are the headline footgun.** react-select renders every filtered option as a React element with no built-in windowing. Lists in the low thousands cause visible input lag on open/type. The v2 line once shipped a `react-virtualized` integration; v3+ removed it. The standard fix is to swap in a virtualized `MenuList` via the components API, or use the community wrapper `react-windowed-select` (react-window under the hood)[^5]. Budget for this before you ship a picker over a big dataset.

**Bundle size.** react-select plus emotion is heavy relative to a plain dropdown — on the order of tens of KB gzipped once emotion is included. If you use it in many small bundles or on latency-sensitive pages, measure it; a headless alternative is often dramatically smaller.

**SSR and CSP.** Because styling is runtime emotion, server rendering needs an emotion cache set up correctly or you get hydration mismatches / flashes of unstyled control. Strict Content-Security-Policy setups must supply a `nonce` to emotion; otherwise injected `<style>` tags are blocked and the control renders unstyled.

**Menu clipping and z-index.** A menu rendered inside a container with `overflow: hidden` or inside modals/tables gets clipped. The fix is `menuPortalTarget={document.body}` plus a `menuPortal` style raising `zIndex`[^6]. This is one of the most-searched react-select issues.

**TypeScript friction.** With `isMulti`, `onChange` receives a readonly array; without it, a single option or `null`. Narrowing this in generic wrapper components is awkward, and the `ActionMeta` argument is easy to ignore. Reusable wrappers usually need to re-declare the three generics.

**Accessibility.** ARIA support is reasonable out of the box (combobox roles, `aria-live` announcements). Custom `components` overrides can silently break keyboard nav and screen-reader output — test with a keyboard and a screen reader after any injection.

**Maintenance cadence.** Treat it as stable-but-slow. Bugs and React-major compatibility fixes can lag; pin versions and test on React upgrades rather than assuming a fast patch.

## When to Use / When Not

**Use when:**
- You need multi-select, async loading, creatable tags, or option groups without building them.
- You want a working, accessible-enough control today and are fine styling via the styles/`classNames` API.
- Your option counts are modest (hundreds, not tens of thousands) or you're willing to add virtualization.

**Avoid when:**
- Bundle size is tight — emotion + the component tree is a real cost.
- You want full control of markup and styling; a headless primitive fits better.
- You render very large lists and don't want to wire up windowing.
- A native `<select>` or a simple custom dropdown already covers your case — don't pull in this much machinery for a 5-item picker.

## Alternatives

- downshift-js/downshift — headless combobox/select primitives; use when you want to own all markup, styling, and a11y decisions.
- radix-ui/primitives — unstyled, accessible Select; use when you need a single-select popover and not async search/multi/creatable.
- tailwindlabs/headlessui — Combobox/Listbox for Tailwind projects; use when you want a lighter, style-it-yourself control.
- mui/material-ui — its Autocomplete covers search/multi/async; use when you're already in the MUI design system.
- JedWatson/react-select via react-windowed-select — same API with virtualization; use when you're committed to react-select but need large lists.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-08 | Repo created; component built for KeystoneJS[^1]. |
| v1 | ~2016 | First widely-adopted release; custom CSS-based styling. |
| v2 | 2018 | Major rewrite backed by Thinkmill/Atlassian: emotion styling, component injection API[^1]. |
| v3 | 2019 | Emotion 10, hooks-era internals; React Conf 2019 talk[^2]. |
| v4 | 2020 | Bundling/API refinements; groundwork for typing. |
| v5 | 2021 | Full TypeScript rewrite; generic `Select<Option, IsMulti, Group>`[^4]. |

## References

[^1]: react-select README — origin in KeystoneJS, funded by Thinkmill and Atlassian. https://github.com/JedWatson/react-select
[^2]: Jed Watson, "Building React Select" — React Conf 2019. https://youtu.be/yS0jUnmBujE
[^3]: emotion — CSS-in-JS library used by react-select for styling. https://emotion.sh
[^4]: react-select docs, TypeScript guide — v5 is a rewrite from JavaScript to TypeScript. https://react-select.com/typescript
[^5]: react-windowed-select — react-window-based virtualization wrapper for react-select. https://github.com/jacobworrel/react-windowed-select
[^6]: react-select docs, advanced usage — `menuPortalTarget` and portal styling. https://react-select.com/advanced

## Tags

react, typescript, javascript, ui-component, select, dropdown, combobox, form-controls, frontend, css-in-js, accessibility
