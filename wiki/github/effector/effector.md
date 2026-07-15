# effector/effector

> A reactive state manager for JavaScript built on an explicit computation graph, with first-class SSR isolation and framework-agnostic core.

[GitHub repo](https://github.com/effector/effector) ·
[Official website](https://effector.dev) ·
[License: MIT](https://github.com/effector/effector/blob/master/LICENSE)

## Overview

Effector is a state management library for JavaScript and TypeScript, first published in 2018 by Dmitry Boldyrev (`zerobias`) and a mostly Russian-speaking contributor group[^1]. Unlike store libraries that expose a single mutable object, Effector models an application's business logic as a directed graph of three primitive "units": events (something happened), stores (state derived from events), and effects (async side effects with their own lifecycle events). You describe how units connect once, statically, and the library's kernel propagates updates through the graph at runtime.

The defining design bet is that logic should live outside the view layer entirely. The `effector` core package has no dependency on any UI framework; thin binding packages (`effector-react`, `effector-vue`, `effector-solid`) subscribe components to stores, and Svelte works with the core directly. This makes the same logic testable in Node with no DOM and portable across React, Vue, Solid, and vanilla apps. The cost is a genuine mental-model shift: there are no reducers, no `dispatch`, no hooks-as-state, and the central connector operator `sample` takes time to internalize.

Effector's other distinguishing feature is its Fork API — isolated computation scopes (`fork`, `allSettled`, `serialize`, `hydrate`) that make server-side rendering and per-test isolation deterministic without global-state leakage[^2]. This is more rigorous than most competitors offer out of the box, and it is also the source of Effector's steepest production footguns.

## Getting Started

```bash
npm install effector
# framework bindings, as needed:
npm install effector-react   # or effector-vue / effector-solid
```

```ts
import { createEvent, createStore, createEffect, sample } from "effector";

// events are inputs; stores are state; effects are async
const buttonClicked = createEvent();
const $count = createStore(0).on(buttonClicked, (n) => n + 1);

const fetchUserFx = createEffect(async (id: number) => {
  const res = await fetch(`/api/user/${id}`);
  return res.json();
});

// `sample` is the universal connector: on click, read $count, run the effect
sample({ clock: buttonClicked, source: $count, target: fetchUserFx });

$count.watch((n) => console.log("count is", n));
```

```tsx
// React binding — useUnit subscribes and returns bound callbacks
import { useUnit } from "effector-react";

function Counter() {
  const [count, onClick] = useUnit([$count, buttonClicked]);
  return <button onClick={onClick}>{count}</button>;
}
```

## Architecture / How It Works

Everything reduces to three unit types plus operators that wire them together. Stores are never mutated directly; they are updated by declaring `.on(event, reducer)` or by being the `target` of a `sample`. `sample` is the workhorse — it reads a `source` when a `clock` fires, optionally filters and transforms, and writes to a `target`. Older operators `forward` and `guard` were deprecated in favor of `sample`, which subsumes both[^3].

Under the hood Effector compiles these declarations into a static graph of nodes and runs them through a single kernel loop. The kernel uses a priority queue with distinct layers (child, pure, sampler, effect) so that updates propagate in a consistent topological order. This is what gives Effector its glitch-free guarantee: in a diamond-shaped dependency (A feeds both B and C, which both feed D), D recomputes once with consistent inputs rather than firing twice with an intermediate value. Achieving that ordering is precisely why the graph must be declared statically rather than assembled ad hoc during rendering.

`combine` derives a store from several stores; `split` routes an event into branches by predicate; `merge` unifies several units into one event stream. `createDomain` groups units for bulk instrumentation, though domains are de-emphasized in current guidance in favor of the Fork API and explicit `sample`.

For SSR and isolation, a `Scope` created by `fork()` holds a private copy of every store's value. `allSettled(event, { scope })` runs a graph to completion and resolves when all triggered effects settle — the basis of deterministic server rendering and integration tests. To make serialization stable across the server/client boundary, every unit needs a Stable IDentifier (SID); these are injected at build time by `effector/babel-plugin` or the SWC plugin. Without SID injection, `serialize`/`hydrate` cannot match units between environments and devtools show anonymous nodes.

## Production Notes

**The babel/SWC plugin is not optional for SSR.** SID generation, readable unit names in devtools, and `serialize`/`hydrate` correctness all depend on it. Teams that skip the plugin ship apps that render fine in dev and then hydrate incorrectly or throw "unit without sid" warnings in production. Configure it before writing any fork-based code.

**Scope leakage is the classic footgun.** Inside a forked scope, any callback that later triggers a unit must be bound with `scopeBind`, otherwise the update escapes to the global scope and silently diverges from the scope's copy of state. The "no scope found" errors and subtle SSR state mismatches almost always trace back to an unbound handler or a missing `{ scope }` argument to `allSettled`.

**React binding version hygiene.** Prefer `useUnit` — the older `useStore`, `useEvent`, and `useList` hooks are deprecated, and mixing scope-aware and scope-unaware hooks in one tree produces inconsistent reads. Wrap the app in `<Provider value={scope}>` from `effector-react/scope` for SSR.

**Bundle size and tree-shaking are strong points.** The core is small (roughly 10–15 kB min+gzip depending on version) and side-effect-free, so unused operators drop out. There is no runtime proxy overhead because there are no proxies.

**Governance and bus factor.** Effector is community-funded (Patreon, OpenCollective, GitHub Sponsors) with a small core team led by one author, not a corporate-backed project. Development is steady but release cadence and issue-triage latency reflect a volunteer maintainer group; the ~160 open issues are typical for a library of this age. English documentation was historically rough and has improved substantially, but some deep guides still read as translations.

## When to Use / When Not

**Use when:**
- Business logic is complex enough that keeping it out of components pays off, and you want it framework-independent and unit-testable in plain Node.
- You need deterministic SSR or per-test state isolation and want it built in rather than hand-rolled.
- You value glitch-free, ordered updates across many interdependent stores (forms, wizards, coordinated async flows).
- You want strong TypeScript inference without decorators, classes, or proxies.

**Avoid when:**
- The app is small; a single Zustand or Context store is less to learn and less ceremony.
- Your state is mostly server cache; TanStack Query solves that more directly than modeling every request as an effect graph.
- The team cannot invest in the `sample`/scope mental model — the learning curve is real and shallow familiarity leads to misuse.
- You need a large corporate support guarantee or a very large hiring pool familiar with the library.

## Alternatives

- pmndrs/zustand — use instead when you want a minimal hook-based store with almost no concepts to learn.
- reduxjs/redux-toolkit — use instead when you want the largest ecosystem, DevTools, and hiring pool despite more boilerplate.
- mobxjs/mobx — use instead when you prefer transparent reactive mutation of observable objects over an explicit graph.
- artalar/reatom — use instead when you want a similarly graph-based, framework-agnostic model with a smaller API surface.
- TanStack/query — use instead when your "state" is really server data fetching, caching, and invalidation.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-02 | First public release; events/stores/effects core[^1]. |
| 20.x | 2019 | API consolidation around units and `createStore`/`createEvent`/`createEffect`. |
| 21.x | 2020 | Bindings maturity; `effector-react` / `effector-vue` alignment. |
| 22.x | 2021–2022 | Fork API / scopes stabilized for SSR and test isolation[^2]. |
| 23.x | 2023 | `forward` and `guard` deprecated in favor of `sample`; `useUnit` becomes the recommended React hook[^3]. |

## References

[^1]: effector README and repository history, effector/effector (created 2018-02-27). https://github.com/effector/effector
[^2]: Effector documentation, "Server Side Rendering" and Fork API (`fork`, `allSettled`, `serialize`, `hydrate`). https://effector.dev/en/api/effector/fork/
[^3]: Effector documentation, `sample` and migration notes deprecating `forward`/`guard`. https://effector.dev/en/api/effector/sample/

## Tags

typescript, javascript, state-management, reactive, event-driven, frontend, react, vue, ssr, business-logic, framework-agnostic
