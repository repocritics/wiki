# developit/mitt

> A ~200-byte functional event emitter / pubsub, with no dependencies and a wildcard listener.

[GitHub repo](https://github.com/developit/mitt) ·
[npm](https://npm.im/mitt) ·
[License: MIT](https://github.com/developit/mitt/blob/main/LICENSE)

## Overview

Mitt is a minimal event emitter written by Jason Miller (developit), also the
author of Preact[^1]. It exists to answer one question — "I need to broadcast
events between decoupled parts of an app, and I don't want to ship the full
Node `EventEmitter` surface or a stream library" — and it answers only that. The
entire implementation is roughly a dozen lines and ships under 200 bytes
gzipped, which is the project's whole selling point and the reason it appears as
a transitive dependency in a large swath of the frontend ecosystem[^2].

The design is deliberately functional: `mitt()` returns a plain object whose
`on`/`off`/`emit` methods do not rely on `this`, so they can be destructured and
passed around freely. There is no class, no inheritance, no `prototype` surface.
Events are keyed in a `Map` from event name (string or symbol) to an array of
handlers, and a special `"*"` wildcard type receives every emitted event.

The defining tension is scope. Mitt gives you `on`, `off`, and `emit` and
nothing else — no `once`, no error isolation, no async, no namespacing, no
handler-count introspection beyond reading the exposed `all` Map yourself. That
minimalism is the feature for library authors counting bytes, and the friction
for application authors who eventually reach for one of those missing pieces and
must either hand-roll it or migrate. Mitt does not try to grow into those needs;
it is effectively feature-complete and changes rarely.

## Getting Started

```sh
npm install --save mitt
```

```js
import mitt from 'mitt'

const emitter = mitt()

emitter.on('foo', e => console.log('foo', e))     // typed listener
emitter.on('*', (type, e) => console.log(type, e)) // wildcard: sees every event

emitter.emit('foo', { a: 'b' })   // -> "foo { a: 'b' }" then "foo { a: 'b' }"

function onFoo() {}
emitter.on('foo', onFoo)
emitter.off('foo', onFoo)         // must pass the SAME reference to remove
emitter.all.clear()              // nuke every listener
```

TypeScript users get key/payload inference by passing an event map. Enable
`"strict": true` in `tsconfig.json` for the tightest checks[^3]:

```ts
import mitt from 'mitt'

type Events = { foo: string; bar?: number }
const emitter = mitt<Events>()

emitter.on('foo', e => {})   // e: string
emitter.emit('foo', 42)      // Error 2345: number not assignable to string
```

## Architecture / How It Works

Mitt is a closure over a single `Map`. `mitt()` optionally accepts an existing
map as its `all` store; otherwise it creates one. Each method operates directly
on that map:

- **`on(type, handler)`** looks up the array for `type`, pushes the handler, or
  creates a new single-element array.
- **`off(type, handler)`** splices the handler out of the array by reference. If
  `handler` is omitted, it replaces the array with an empty one — removing every
  listener for that type at once.
- **`emit(type, evt)`** clones the handler array (`slice()`) and calls each
  handler synchronously with `evt`, then calls each `"*"` handler with
  `(type, evt)`. The wildcard handler signature therefore differs from a typed
  handler's.
- **`all`** is the Map itself, exposed directly. There is no encapsulation —
  reading listener counts or clearing everything (`all.clear()`) is done by
  touching the Map.

The switch to a `Map` (from a plain object in earlier versions) is the one
notable internal change in the project's history. Using a `Map` allows `symbol`
event keys and sidesteps prototype-pollution and reserved-key hazards that a
plain-object store would expose. Handlers are invoked in registration order.

There is no scheduling, batching, or microtask deferral: `emit` is a
straight-line synchronous loop. This is what keeps the byte count low and makes
behavior trivially predictable, but it also means the caller's stack and the
handler's stack are the same stack.

## Production Notes

The footguns all follow from the "does exactly three things" philosophy:

- **No error isolation.** `emit` does not wrap handlers in `try/catch`. A single
  throwing listener aborts the loop, so later handlers — including the `"*"`
  handler — never run. If you attach listeners from untrusted or independent
  modules, wrap your own handler bodies.
- **Synchronous, same-stack execution.** Handlers run inline during `emit`. A
  slow handler blocks the emitter; a handler that re-emits can recurse. There is
  no async variant.
- **No `once`.** You must `off` yourself inside the handler, which means you need
  a named reference — inline arrow handlers cannot be removed. This is the most
  common reason teams outgrow mitt.
- **Removal is by reference.** `off('foo', fn)` only works if `fn` is the exact
  function passed to `on`. Bound or wrapped functions won't match, a frequent
  source of "my listener won't detach" leaks in component lifecycles.
- **`off(type)` with no handler wipes the whole type.** Convenient, but easy to
  do accidentally and remove listeners you didn't own.
- **Leaks are your responsibility.** Handlers live in the Map until explicitly
  removed. In frameworks, pair every `on` in mount with an `off` in unmount.
- **Only literal `"*"`.** There is no glob or namespace matching
  (`"user:*"`); the wildcard is all-or-one.
- **Manually emitting `"*"` is unsupported** — the wildcard is a fan-out sink,
  not an emittable type.

The project is stable to the point of dormancy: the last publish and most commit
activity predate 2024, and issues/PRs accumulate slowly[^2]. For a library this
small and this finished, low churn is a reasonable state rather than
abandonment, but do not expect new features or fast issue turnaround.

## When to Use / When Not

**Use when:**
- You are a library author who needs a decoupled event bus and every kilobyte in
  your bundle is scrutinized.
- You want a `this`-free, destructurable emitter you can pass around functionally.
- Your event flow is simple: fire, subscribe, unsubscribe, occasionally listen to
  everything for logging/debugging.

**Avoid when:**
- You need `once`, error isolation, async/awaitable emit, or wildcard patterns —
  you'll reimplement them on top and lose the size advantage.
- You want backpressure, operators, or composition — that's a streams problem
  (RxJS), not an emitter problem.
- You're in Node and already have the built-in `events` module with a richer API
  and no install cost.

## Alternatives

- primus/eventemitter3 — use when you want a fast, `once`-capable, Node-style
  `EventEmitter` API in the browser and can spend a few more kilobytes.
- ai/nanoevents — use when you want mitt-level minimalism with a slightly
  different unsubscribe-return ergonomics.
- nodejs/node (`events`) — use when you're server-side and want the standard
  library emitter with no dependency at all.
- ReactiveX/rxjs — use when events are really streams and you need operators,
  combination, and backpressure, accepting a much larger footprint.
- Morglod/tseep — use when emit throughput is a measured bottleneck and you want
  a high-performance emitter with a familiar API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017-01 | Initial release: tiny functional emitter, plain-object store[^1]. |
| 2.0 | 2019 | API refinements; still object-backed event map. |
| 3.0 | 2022 | Rewrite to a `Map`-backed store; symbol event keys, sharper TypeScript generics[^3]. |

(Patch releases within the 3.x line followed; exact patch dates are omitted where
not independently confirmed.)

## References

[^1]: mitt on npm and repository, Jason Miller (developit). https://npm.im/mitt
[^2]: developit/mitt repository — commit/release activity and issue tracker. https://github.com/developit/mitt
[^3]: mitt README, "Typescript" section — event-map generics and `Emitter` type. https://github.com/developit/mitt#typescript

## Tags

javascript, typescript, event-emitter, pubsub, event-bus, frontend, browser, tiny-library, zero-dependency, mit-license
