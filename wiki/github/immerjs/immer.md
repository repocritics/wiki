# immerjs/immer

> Produce the next immutable state by writing ordinary mutating code against a proxied draft.

[GitHub repo](https://github.com/immerjs/immer) ·
[Official website](https://immerjs.github.io/immer/) ·
[License: MIT](https://github.com/immerjs/immer/blob/main/LICENSE)

## Overview

Immer is a small library for producing immutable state updates in JavaScript. Instead of hand-writing spread operators and nested `Object.assign` calls, you mutate a *draft* of the current state inside a recipe function, and Immer computes the next immutable state for you. It was created by Michel Weststrate (also the author of MobX) and open-sourced in late 2017[^1]. It won the React ecosystem's "Breakthrough of the year" open-source award in 2019[^2].

The reason Immer is one of the most-depended-upon packages in the JavaScript state-management space is not direct adoption — it is that Redux Toolkit embeds Immer in every `createSlice` / `createReducer` call[^3]. Any project using the modern Redux happy path is using Immer transitively, which is why the download numbers dwarf the ~29k stars on this repo. With roughly 29,000 stars and a last push in mid-2026, the project is mature and maintained but no longer under heavy feature development; the API surface has been stable since the v10 line.

The defining tradeoff: Immer trades runtime cost for authoring ergonomics. Every draft access goes through a `Proxy`, and by default every produced state is deep-frozen. For most application state this is invisible; for very large collections or hot update loops it is measurable, and the honest advice from the docs themselves is that hand-written reducers are faster when it matters[^4].

## Getting Started

```bash
npm install immer
```

```js
import { produce } from "immer";

const base = {
  user: { name: "Ada", tags: ["admin"] },
  count: 1,
};

const next = produce(base, (draft) => {
  draft.count++;              // ordinary mutation
  draft.user.tags.push("beta");
});

next !== base;               // true  — new object
next.user !== base.user;     // true  — changed branch is copied
next.count;                  // 2
base.count;                  // 1     — original untouched
```

`produce` can also be curried into a reducer by omitting the base state: `produce((draft, action) => { ... })` returns a function `(state, action) => nextState`.

## Architecture / How It Works

The core is copy-on-write over ES `Proxy` objects. When you call `produce(base, recipe)`, Immer wraps `base` in a proxy and passes it as the draft. Reads pass through to the original. The first write to any node triggers a shallow copy of that node (and, lazily, of its ancestors up to the root); further writes hit the copy. Nodes you never touch are never copied, so the result **structurally shares** all unchanged subtrees with the original — this is what makes downstream referential-equality checks (`React.memo`, `useMemo`, Redux selectors) cheap.

At the end of the recipe Immer *finalizes*: it walks the drafted tree, swaps proxies for their concrete copies, and — unless disabled — calls `Object.freeze` recursively on the new parts (**auto-freeze**). Auto-freeze is what turns "you should treat state as immutable" into "the runtime enforces it," and it is on by default.

Immer is deliberately modular. Several capabilities are opt-in plugins you must enable once at startup:

- `enableMapSet()` — draft support for native `Map` and `Set` (not enabled by default).
- `enablePatches()` — `produceWithPatches` returns JSON-patch and inverse-patch arrays alongside the next state, the basis for undo/redo and change-syncing.

Other primitives: `current(draft)` gives a plain snapshot of a draft mid-recipe; `original(draft)` returns the corresponding node from the base state; the `immerable` symbol marks class instances as safe to draft; `castDraft` / `castImmutable` are type-only helpers for TypeScript. A recipe either mutates the draft *or* returns a brand-new value — doing both throws, because Immer cannot reconcile the two intents.

## Production Notes

**Auto-freeze cost.** Because auto-freeze recurses over the *entire* produced state, feeding Immer a large object where you only change one leaf can still cost a full deep-freeze the first time that object passes through `produce`. If profiling shows freezing dominating, `setAutoFreeze(false)` disables it globally — but then nothing stops accidental mutation of "immutable" state, so this is a knowing tradeoff, not a free win.

**Proxy overhead.** Draft reads and writes are not free. In tight loops (e.g. building state for thousands of items), a plain reducer with spreads, or a purpose-built structure, will outperform Immer. The library is designed for the common case of moderate-sized application state, not for being the fastest possible path.

**Plugins are silent until enabled.** A very common bug: mutating a `Map` or `Set` in a draft without having called `enableMapSet()` — the values are treated as opaque and your changes silently do not take. Enable the plugins once, at module load, before any `produce` runs.

**Class instances and non-plain data.** Immer only drafts plain objects, arrays, and (with the plugin) Maps/Sets. Class instances are copied by reference unless the class carries the `[immerable] = true` marker. `Date`, typed arrays, DOM nodes, and other exotic objects are treated as atomic — replace them wholesale rather than mutating in place.

**Don't leak drafts.** A draft is only valid inside its recipe. Storing a reference to a draft (or a piece of one) and using it after `produce` returns is undefined behavior; use `current()` if you need a durable snapshot mid-recipe.

**Upgrade note (v10).** Immer 10 removed the legacy ES5 fallback and the default export; code importing `immer` as a default, or relying on `enableES5()` for pre-Proxy environments, must migrate to named imports and a Proxy-capable runtime[^5]. In practice this only affects very old targets — all evergreen browsers and supported Node versions have `Proxy`.

## When to Use / When Not

**Use when:**
- You have nested, non-trivial immutable state and want to stop writing spread pyramids.
- You are already on Redux Toolkit (you're using Immer regardless — lean into it).
- You need patch generation for undo/redo or state syncing (`enablePatches`).
- Correctness-by-default matters more than the last few percent of update throughput.

**Avoid when:**
- The state update is a hot path over large collections — measure; a hand-written reducer or a faster drop-in may be warranted.
- You want dedicated persistent data structures with their own API (lookups, lazy sequences) rather than frozen plain objects.
- You prefer a reactive/mutable model over immutable snapshots.

## Alternatives

- unadlib/mutative — near-drop-in API that markets itself as substantially faster than Immer; use when Immer's proxy/freeze overhead is a *measured* bottleneck.
- immutable-js/immutable-js — persistent collections (List, Map, Record) with their own types; use when you want genuine immutable data structures rather than frozen plain JS, and can afford the to/from-JS conversion boundary.
- mobxjs/mobx — same author, opposite philosophy: observable mutable state; use when you want reactivity instead of immutable snapshots.
- kolodny/immutability-helper — small command-based `update()` helper; use when you want tiny, explicit updates without Proxy semantics or a freeze pass.
- reduxjs/redux-toolkit — bundles Immer already; if you're building Redux state, use it directly rather than adding Immer as a separate dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018 | First release; Proxy-based `produce`, structural sharing[^1]. |
| 6.0 | 2020-03 | Modular plugin rewrite (MapSet / Patches / ES5), tree-shakeable, auto-freeze on by default. |
| 9.0 | 2021 | API refinements; stricter draft handling. |
| 10.0 | 2023 | Dropped the ES5 fallback and default export; Proxy-only, ESM-first; performance work[^5]. |

## References

[^1]: Immer documentation and repository — Michel Weststrate, creator (also author of MobX). https://immerjs.github.io/immer/
[^2]: React Open Source Awards 2019, "Breakthrough of the year" (cited in the project README). https://osawards.com/react/
[^3]: Redux Toolkit — "Writing Reducers with Immer." https://redux-toolkit.js.org/usage/immer-reducers
[^4]: Immer documentation — "Performance." https://immerjs.github.io/immer/performance/
[^5]: Immer release notes. https://github.com/immerjs/immer/releases

## Tags

javascript, typescript, immutability, state-management, redux, proxy, structural-sharing, copy-on-write, react, frontend
