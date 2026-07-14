# statelyai/xstate

> State machines, statecharts, and the actor model for JavaScript/TypeScript — model application logic explicitly instead of scattering it across booleans.

[GitHub repo](https://github.com/statelyai/xstate) ·
[Official website](https://stately.ai/docs) ·
[License: MIT](https://github.com/statelyai/xstate/blob/main/LICENSE)

## Overview

XState is a zero-dependency library for modeling logic as finite state machines, statecharts, and actors[^1]. Instead of tracking application state with a scatter of boolean flags (`isLoading`, `isError`, `isSubmitting`) that can drift into impossible combinations, you declare the finite set of states a system can be in, the events that move between them, and the side effects each transition triggers. The formalism is David Harel's statecharts (1987) and the W3C SCXML specification, both of which XState follows closely[^2].

It is maintained by Stately (statelyai), the company founded by original author David Khourshid around the library. Stately's commercial product is a visual editor (Stately Studio / state.new) that reads and writes XState machines — the library is the open-source substrate. This is the defining tension: XState is genuinely framework-agnostic and runs anywhere, but its most polished authoring experience is tied to a hosted visual tool, and the library's biggest conceptual competitor is "just use `useState`."

XState's second tension is weight versus expressiveness. The full statechart model (hierarchy, parallelism, history, delayed transitions, spawned actors) is more machinery than most component-level state needs. Stately's own answer to this is `@xstate/store`, a sub-1kb event-based store shipped from the same monorepo for cases where you want predictable updates without the machine formalism[^3].

## Getting Started

```bash
npm install xstate
```

```ts
import { createMachine, createActor, assign } from 'xstate';

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  context: { count: 0 },
  states: {
    inactive: {
      on: { TOGGLE: { target: 'active' } }
    },
    active: {
      entry: assign({ count: ({ context }) => context.count + 1 }),
      on: { TOGGLE: { target: 'inactive' } }
    }
  }
});

// An actor is a running instance of the machine (like a store).
const actor = createActor(toggleMachine);
actor.subscribe((snapshot) => console.log(snapshot.value, snapshot.context));
actor.start();                    // => 'inactive' { count: 0 }
actor.send({ type: 'TOGGLE' });   // => 'active'   { count: 1 }
actor.send({ type: 'TOGGLE' });   // => 'inactive' { count: 1 }
```

Framework bindings live in separate packages: `@xstate/react`, `@xstate/vue`, `@xstate/svelte`, `@xstate/solid`. In React, `useMachine`/`useActor` wire a machine into a component and re-render on snapshot changes.

## Architecture / How It Works

The core model has four pieces:

- **Machine** — a pure, serializable definition: `states`, transitions (`on`), extended state (`context`), and the `actions`, `guards`, and `actors` a transition may reference. A machine is a value, not a running thing; it can be inspected, visualized, and diffed.
- **Actor** — a running instance created by `createActor(machine)`. It holds current state, receives events via `.send()`, exposes `.getSnapshot()`, and notifies subscribers. Actors can spawn and communicate with child actors, which is how XState models concurrency.
- **Actions** — side effects fired on entry, exit, or transition. `assign` is the special action that updates `context` immutably. Actions are described in the machine but executed by the actor, keeping the definition pure.
- **Guards** — boolean predicates (`guard` in v5, `cond` in v4) that gate whether a transition is taken.

Statechart features beyond flat FSMs: **hierarchical (nested) states** to factor shared transitions, **parallel states** (`type: 'parallel'`) for orthogonal regions active simultaneously, **history states** to resume a region where it left off, **final states**, **delayed transitions** (`after`), and **eventless/always transitions**. Long-running work is modeled with `invoke` (promises, callbacks, observables, or other machines) whose lifecycle is bound to the invoking state.

The v5 rewrite (2023) reframed everything around the actor model: `interpret` became `createActor`, services became actors, and a new `setup()` API centralizes the typed `actions`/`guards`/`actors` a machine uses. Because the machine is a plain object graph, the same definition feeds the interpreter, the graph-traversal utilities (`@xstate/graph`), model-based test generation, and Stately's visualizer — one source of truth rendered many ways.

## Production Notes

**The v4 → v5 migration is large.** v5 (December 2023) changed enough surface area that upgrading a non-trivial v4 codebase is a real project, not a version bump: `interpret` → `createActor`, `Machine`/`createMachine` config changes, `cond` → `guard`, `services` → `actors`, changes to `send`/`sendParent`/`spawn`, and the introduction of `setup()`. Many teams stayed on v4 for a long time; treat a v4→v5 move as a planned migration with the official guide open[^4].

**TypeScript inference was historically the weak point.** In v4, getting events and context to type correctly across actions often required the separate `@xstate/typegen` codegen step, which added a build-time dependency and its own footguns (stale generated files, editor integration). v5's `setup()` API was designed to make inference work without typegen, and typegen is deprecated for v5. If you're evaluating XState on old tutorials, know that the DX story changed materially at v5.

**Bundle size and conceptual weight.** Core `xstate` is meaningfully larger than a minimal store and pulls in the whole statechart engine. For a single toggle or a small global store, the machinery is overkill — this is exactly why `@xstate/store` exists. Reach for full machines when the logic has genuine states, guards, and concurrent effects; reach for a store (or plain `useState`) when it doesn't.

**Determinism and testability are the real payoff.** Because transitions are pure and the machine is serializable, you get exhaustive model-based testing (generate test paths from the graph), replayable event logs, and a visual diff of behavior. Teams that adopt XState successfully tend to be the ones that exploit this — not the ones using it as a fancier reducer.

**SCXML compatibility is partial.** XState is inspired by and largely aligned with SCXML but is not a complete implementation; don't assume every SCXML construct round-trips.

## When to Use / When Not

**Use when:**
- Logic has clearly enumerable states with rules about what transitions are legal (wizards, checkout/payment flows, connection lifecycles, media players, auth).
- You want impossible states to be unrepresentable rather than defended against with flag combinations.
- You benefit from visualization, model-based testing, or handing a diagram to non-engineers.
- You're orchestrating long-running or concurrent effects (invoked promises, spawned child actors).

**Avoid when:**
- The state is a handful of independent values with no meaningful transition rules — `useState`/`useReducer` or a small store is less ceremony.
- You need a plain global data store (server cache, form values) — that's a store or data-fetching library's job, not a statechart's.
- The team won't invest in learning statecharts; misapplied XState becomes an obfuscated reducer.
- Bundle size is critical and the logic is trivial — consider `@xstate/store` or nothing.

## Alternatives

- pmndrs/zustand — minimal hook-based store; use instead when you want simple shared state without the machine formalism.
- reduxjs/redux — predictable global store with a large ecosystem; use when you need centralized, middleware-heavy app state rather than per-flow logic.
- mobxjs/mobx — transparent reactive state; use when you prefer observable mutation over explicit events and transitions.
- statelyai/xstate's own `@xstate/store` — same repo, sub-1kb event store; use for simple predictable updates when full machines are too much.
- Robot (thisrobot/robot) — much smaller functional FSM library; use when you want basic finite state machines with a tiny footprint and no statechart features.

## History

| Version | Date | Notes |
|---------|------|-------|
| 3.x | 2018 | Early statechart implementation, gained traction in the React community. |
| 4.0 | 2019 | Long-lived stable major; the version most tutorials and production code targeted for years. |
| 4.x | 2020–2023 | `@xstate/react` hooks, `@xstate/typegen` for TS inference, model-based testing tooling. |
| 5.0 | 2023-12 | Actor-model rewrite: `createActor`, `setup()` API, guards/actors renaming, typegen deprecated[^4]. |
| `@xstate/store` | 2024 | Standalone sub-1kb event store for simple state, shipped from the monorepo[^3]. |

## References

[^1]: XState README, statelyai/xstate. https://github.com/statelyai/xstate
[^2]: David Harel, "Statecharts: A Visual Formalism for Complex Systems" (1987), and the W3C SCXML specification. https://www.w3.org/TR/scxml/
[^3]: `@xstate/store` package. https://github.com/statelyai/xstate/tree/main/packages/xstate-store
[^4]: XState v5 migration guide, Stately docs. https://stately.ai/docs/migration

## Tags

typescript, javascript, state-machine, statechart, finite-state-machine, actor-model, state-management, orchestration, frontend, scxml
