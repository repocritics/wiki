# downshift-js/downshift

> Headless primitives for building WAI-ARIA compliant autocomplete, combobox, and select components in React — you own all the rendering.

[GitHub repo](https://github.com/downshift-js/downshift) ·
[Official website](https://downshift-js.com/) ·
[License: MIT](https://github.com/downshift-js/downshift/blob/master/LICENSE)

## Overview

Downshift is a headless component library for the narrow but perennially hard
problem of accessible select/combobox/autocomplete widgets. It ships no markup
and no styles: it manages the state machine (open/closed, highlighted index,
input value, selection) and the ARIA wiring, and hands you *prop getters* —
functions like `getInputProps()`, `getMenuProps()`, `getItemProps()` — that you
spread onto the elements you render yourself. It was created by Kent C. Dodds in
2017 and popularized the "prop getter" pattern that later spread across the
React headless-UI ecosystem[^1].

The library has two generations of API living side by side. The original is the
`Downshift` render-prop component; the current, recommended surface is a set of
hooks — `useSelect`, `useCombobox`, and the newer `useTagGroup`. The maintainers
are explicit that the hooks are where work happens and that the `Downshift`
component will eventually be removed once the hooks reach feature parity[^2].
Only the hooks implement the current W3C ARIA 1.2 combobox pattern; the legacy
component does not, which is the single most important fact for anyone choosing
between them today.

The defining tradeoff is control versus batteries. Downshift gives you total
freedom over DOM structure, styling, filtering, and virtualization at the cost
of writing that rendering (and the accessibility-correct HTML structure)
yourself. Teams that want a dropdown that "just renders" are usually happier
with a batteries-included library; teams that need a bespoke widget that is
still screen-reader-correct are exactly who Downshift is for. It is broadly
adopted (~12k stars, ~940 forks) and remains actively maintained[^3].

## Getting Started

```bash
npm install --save downshift   # peer dependency: react
```

```jsx
import {useCombobox} from 'downshift'

const items = ['apple', 'pear', 'orange', 'grape', 'banana']

function FruitCombobox() {
  const [list, setList] = React.useState(items)
  const {
    isOpen,
    getLabelProps,
    getMenuProps,
    getInputProps,
    getItemProps,
    highlightedIndex,
  } = useCombobox({
    items: list,
    onInputValueChange: ({inputValue}) =>
      setList(items.filter(i => i.includes(inputValue ?? ''))),
  })

  return (
    <div>
      <label {...getLabelProps()}>Choose a fruit</label>
      <input {...getInputProps()} />
      <ul {...getMenuProps()}>
        {isOpen &&
          list.map((item, index) => (
            <li
              key={item}
              {...getItemProps({item, index})}
              style={{background: highlightedIndex === index ? '#eee' : '#fff'}}
            >
              {item}
            </li>
          ))}
      </ul>
    </div>
  )
}
```

## Architecture / How It Works

At the core is a small reducer-driven state machine. Every user interaction
(keydown, click, blur, input) produces a change of a known `type` — the
`stateChangeTypes` enum — and Downshift computes the next state from it. The
public API is deliberately built around three interception points:

- **Prop getters** — `getInputProps`, `getMenuProps`, `getItemProps`,
  `getToggleButtonProps`, `getLabelProps` return the ARIA attributes, ids, and
  event handlers each element needs. You can pass your own handlers through them;
  Downshift composes rather than overwrites, so `getInputProps({onBlur})` runs
  both its handler and yours.
- **`stateReducer(state, changes)`** — called on every internal state
  transition, letting you veto or rewrite a change (the canonical example is
  keeping the menu open after selection). It must be a pure function.
- **Control props** — passing `isOpen`, `inputValue`, `highlightedIndex`, or
  `selectedItem` as props converts that slice of state to controlled mode, the
  same escape hatch React uses for form inputs.

The hooks (`useSelect`/`useCombobox`) reimplement this machine against the
ARIA 1.2 combobox spec, which changed which element owns focus and how
`aria-activedescendant` is managed versus the older 1.1 pattern the legacy
component encodes[^2]. That spec divergence is why the two APIs cannot be made
identical and why the component is on a deprecation path rather than being
retrofitted.

Downshift ships as UMD, CJS, and ESM builds and works with Preact via a
dedicated entry point (`downshift/preact`). It has no dependency on a positioning
library — anchoring the menu (Popper/Floating UI) is left to you.

## Production Notes

- **Accessibility is not automatic — the HTML structure is.** The `getRootProps`
  wrapper exists specifically because a fully screen-reader-correct combobox
  needs a particular element nesting; the README's "simpler" example without it
  is explicitly documented as *not* fully accessible[^3]. If you deviate from
  the prescribed structure you can ship something that passes visually and fails
  with assistive tech.
- **`itemToString` must null-check.** It is invoked with `null` when the user
  clears input via Escape; a naive `item => item.value` throws in normal use[^3].
- **Legacy vs current API is a real fork in the road.** New code should start on
  the hooks. The `Downshift` render-prop component still works but does not track
  the current ARIA pattern and receives no new features. Migrating an existing
  render-prop integration to hooks is a rewrite of the render function, not a
  drop-in.
- **v7 was a breaking change for hook users.** The migration to the ARIA 1.2
  pattern in version 7 altered behavior and markup expectations; there is an
  official migration guide, and upgrading across it is not transparent[^2].
- **SSR id stability.** Server-rendered items rely on generated ids that must
  match between server and client; Downshift exposes `id`/`resetIdCounter`
  controls for this, and mismatches surface as hydration warnings.
- **Filtering, sorting, async loading, and virtualization are yours.** Downshift
  does not fetch, debounce, or window results — pair it with something like
  react-virtualized/react-window (via the `itemCount` prop) for long lists.

## When to Use / When Not

**Use when:**
- You need a combobox/select/autocomplete with custom DOM and styling but
  correct WAI-ARIA behavior.
- You want a controllable state machine (`stateReducer`, control props) rather
  than a black-box widget.
- You are building a design-system primitive that other components wrap.

**Avoid when:**
- You want a styled, ready-to-drop-in `<Select>` — the render-it-yourself model
  is overhead you do not need.
- You need multi-framework or non-React support — Downshift is React/Preact only.
- Your team is unlikely to test with a screen reader; the freedom Downshift
  gives is also freedom to build an inaccessible widget.

## Alternatives

- JedWatson/react-select — batteries-included, styled, feature-heavy select; use
  it when you want a working dropdown out of the box instead of a headless kit.
- tailwindlabs/headlessui — headless Combobox/Listbox that still renders its own
  elements; use it when you want less boilerplate than prop getters and are on
  React or Vue.
- ariakit/ariakit — broad accessible component system including Combobox; use it
  when you need more than just select/combobox primitives.
- radix-ui/primitives — composable accessible Select/Combobox primitives; use it
  when you are standardizing on Radix for a design system.
- pacocoursey/cmdk — command-palette combobox; use it when you specifically want
  a ⌘K menu rather than a general form control.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | Initial release; render-prop `Downshift` component, prop-getter pattern[^1]. |
| 2.0 | 2018 | API cleanup and control-prop refinements. |
| 3.0 | 2019 | Continued render-prop era; groundwork before hooks. |
| 4.0 | 2020 | `useSelect` / `useCombobox` hooks introduced as the recommended API. |
| 5.0 | 2020 | Hook API stabilization. |
| 6.0 | 2021 | TypeScript and hook improvements. |
| 7.0 | 2022 | Hooks migrated to the W3C ARIA 1.2 combobox pattern; breaking, with migration guide[^2]. |

## References

[^1]: Kent C. Dodds, "Introducing Downshift for React" — introduces the render-prop / prop-getter approach. https://kentcdodds.com/blog/introducing-downshift-for-react
[^2]: Downshift documentation and v7 migration guide — hooks implement the ARIA 1.2 combobox pattern; the `Downshift` component does not and is slated for removal. https://downshift-js.com/
[^3]: downshift-js/downshift README — API surface (`useSelect`, `useCombobox`, `useTagGroup`, `Downshift`), `getRootProps` accessibility note, and `itemToString` null-check caveat. https://github.com/downshift-js/downshift

## Tags

react, combobox, autocomplete, select, dropdown, accessibility, wai-aria, headless-ui, hooks, javascript
