# mobxjs/mobx

> Transparent reactive state management: mutate plain objects, and everything derived from them updates automatically.

[GitHub repo](https://github.com/mobxjs/mobx) ·
[Official website](https://mobx.js.org) ·
[License: MIT](https://github.com/mobxjs/mobx/blob/main/LICENSE)

## Overview

MobX is a state-management library built around transparent functional reactive programming (TFRP). You mark some state as observable, write plain imperative code that mutates it, and MobX tracks — at runtime — exactly which computations and side effects read which fields, then re-runs only those when the fields change. It was created by Michel Weststrate, first released in 2015 under the name Mobservable and renamed MobX the following year[^1]. Development has long been backed by Mendix, where it was proven on a large production codebase[^2].

Its defining tradeoff is the inverse of Redux's. Redux is explicit, immutable, and verbose; MobX is implicit, mutable, and terse. You do not write reducers, action types, or selectors — a normal assignment (`this.count += 1`) is the update. The cost is that the dependency graph is invisible: reactivity works through a runtime observation mechanism that only tracks reads happening *inside* a reactive context (an `observer` component, a `computed`, or a `reaction`). Read an observable outside that context, or destructure it too early, and the tracking silently does not happen. Most MobX bugs are variations on this one theme.

MobX is UI-framework-agnostic at its core; the popular pairing is `mobx-react-lite` / `mobx-react` for React, but the reactive engine runs standalone in Node, Vue, Angular, or plain DOM code. As of 2026 it remains widely used and maintained, though newer minimal stores (Zustand, Jotai, Valtio) have absorbed much of the mindshare for new React projects.

## Getting Started

```bash
npm install mobx mobx-react-lite   # React bindings; use mobx-react for class components
```

```tsx
import { makeAutoObservable } from "mobx";
import { observer } from "mobx-react-lite";

class Timer {
  secondsPassed = 0;
  constructor() {
    makeAutoObservable(this); // fields → observable, methods → actions, getters → computed
  }
  increase() { this.secondsPassed += 1; } // a normal mutation; no setState
  reset() { this.secondsPassed = 0; }
}

const timer = new Timer();
setInterval(() => timer.increase(), 1000);

// observer() subscribes the component to exactly the observables it reads while rendering
const TimerView = observer(() => (
  <button onClick={() => timer.reset()}>Seconds passed: {timer.secondsPassed}</button>
));
```

The `observer` wrapper detects that `TimerView` read `timer.secondsPassed` during render and re-renders it only when *that* field changes — no memoization, selectors, or dependency arrays.

## Architecture / How It Works

MobX has four primitives: **observables** (tracked state), **computeds** (pure values derived from observables, cached and lazily recomputed), **actions** (functions that mutate observables), and **reactions** (side effects — `autorun`, `reaction`, `when`, and the `observer` HOC — that re-run when their tracked reads change).

Internally, every observable field is an `Atom` (or `ObservableValue`). Every reactive context is a `Derivation`. When a derivation runs, MobX records every atom read into that derivation's dependency set and, symmetrically, registers the derivation as an observer of each atom. On mutation, the atom notifies its observers. Propagation is **glitch-free and synchronous**: MobX topologically settles the graph before running side effects, so a reaction never observes a half-updated intermediate state, and each derivation runs at most once per transaction. Computeds are lazy — an unobserved computed is not kept alive and is recomputed on next read rather than eagerly maintained.

The most consequential architectural decision is the shift in how observables are created. Through MobX 5, the idiomatic API was ES decorators (`@observable`, `@computed`, `@action`), which required Babel/TypeScript decorator configuration that was perpetually in flux. **MobX 6 (2020) made decorators optional and introduced `makeObservable` / `makeAutoObservable`**, which annotate an instance from inside the constructor with no build-tool setup[^3]. Decorators still work but are no longer the recommended path. This is the single biggest source of version confusion: tutorials and Stack Overflow answers are split across the two eras.

A second fork exists in the runtime substrate. **MobX 5 rewrote the core on ES6 Proxies**, which lets observable arrays and plain-object maps behave like their native counterparts and drops the old `.get()/.set()` ceremony; **MobX 4 is the Proxy-free branch** kept alive for environments without Proxy support (legacy IE, some React Native JS engines)[^4]. MobX 6 unified the API surface on top of the Proxy core.

## Production Notes

**Reactivity is lost by dereferencing too early.** `observer` tracks only observables *read during render*. Destructuring (`const { secondsPassed } = timer`) outside the component, passing a primitive down as a prop, or reading in a non-reactive callback captures a snapshot value and severs tracking. The rule: pass the observable object and read the leaf field inside the observed render. This is the number-one MobX support question.

**Actions and async.** Under `enforceActions: "observed"` (or `"always"`), mutating observed state outside an action throws. Async functions are a trap: only the synchronous portion runs inside the action, so mutations after an `await` must be wrapped in `runInAction(() => { ... })` or delegated to a separate action. `flow` (generator-based) is the alternative for cancellable async.

**`makeAutoObservable` limitations.** It cannot be used on classes with a superclass, and every field/method must be inferrable; subclassing requires the more explicit `makeObservable` with per-member annotations. Getters become `computed` automatically, which is usually right but occasionally surprising for expensive getters that should be plain.

**Computed lifecycle.** A computed is only cached while it is being observed by a reaction/observer. Read a computed from outside any reactive context and it recomputes every time (correct, but a silent perf cost). Keeping one warm outside the UI requires `keepAlive` or an `autorun` holding it.

**React integration.** Use `mobx-react-lite` for function components (hooks); `mobx-react` layers class-component support on top. Recent versions rely on `useSyncExternalStore`, which resolves the tearing concerns under React 18 concurrent features, but mixing `observer` with `React.memo`/manual memoization can reintroduce stale-closure bugs. Do not select-and-destructure MobX state into local `useState`.

**Synchronous propagation cascades.** Because reactions fire synchronously on mutation, a chain of interdependent reactions can run mid-mutation-batch in ways that surprise. Wrap multi-field updates in `action`/`runInAction` so they settle as one transaction rather than firing observers per assignment.

**Migration.** The v5→v6 move is mostly additive (add `makeObservable` in constructors; decorators keep working), but codebases that relied on decorator behavior, `decorate()`, or implicit observability of subclass fields need review. There is no automatic codemod for the decorator-to-`makeObservable` switch.

## When to Use / When Not

**Use when:**
- Your domain state is a graph of objects with rich derived values, and you want derivations to stay correct automatically.
- You prefer mutable, OOP-style stores and minimal boilerplate over reducer ceremony.
- You want state that lives outside React and is portable/testable on its own.
- Fine-grained, precise re-rendering matters and you don't want to hand-tune memoization.

**Avoid when:**
- Your team values an explicit, auditable, serializable update log (Redux-style time-travel, strict unidirectional flow).
- The app is small enough that a minimal hook store (Zustand/Jotai) covers it with less conceptual surface.
- You need every state transition to be a pure, replayable data structure (event sourcing, undo via snapshots is easier with immutable stores).
- Contributors are junior to reactive tracking; the "lost reactivity" footguns have a real learning curve.

## Alternatives

- reduxjs/redux — explicit immutable single store; choose it when auditability, time-travel, and disciplined unidirectional flow outweigh boilerplate.
- pmndrs/zustand — minimal hook-based store with no provider; choose it for small-to-mid React apps that want less magic and a tiny bundle.
- pmndrs/valtio — proxy-based mutable state with MobX-like ergonomics but a far smaller API; choose it when you like MobX's mutate-and-react feel scoped to React.
- pmndrs/jotai — bottom-up atomic state; choose it when state decomposes into many small independent atoms rather than object graphs.
- statelyai/xstate — statecharts/state machines; choose it when the hard part is complex finite-state logic, not derived data.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 (Mobservable) | 2015 | Initial release of the reactive core[^1]. |
| 2.0 | 2016 | Renamed to MobX[^1]. |
| 3.0 | 2017 | API consolidation, improved TypeScript support. |
| 4.0 | 2018 | Proxy-free branch; maintained for legacy/no-Proxy environments[^4]. |
| 5.0 | 2018 | Core rewritten on ES6 Proxies; native-like observable arrays/objects[^4]. |
| 6.0 | 2020-10 | Decorators made optional; `makeObservable` / `makeAutoObservable` introduced[^3]. |

## References

[^1]: MobX documentation, "About this documentation" and project history. https://mobx.js.org/README.html
[^2]: MobX README, "Credits" — Mendix's role in maintaining MobX. https://github.com/mobxjs/mobx#credits
[^3]: MobX docs, "Enabling decorators (optional)" and `makeObservable` reference — decorators optional since MobX 6. https://mobx.js.org/enabling-decorators.html
[^4]: MobX docs, "Installation" — MobX 5 requires Proxy support; MobX 4 runs on any ES5 environment. https://mobx.js.org/installation.html

## Tags

typescript, javascript, state-management, reactive-programming, frontend, react, observable, mobx, functional-reactive-programming, client-side
