# Effect-TS/effect

> A TypeScript standard library built around a single lazy `Effect<A, E, R>` type that tracks success, typed errors, and dependencies in the signature.

[GitHub repo](https://github.com/Effect-TS/effect) ·
[Official website](https://effect.website) ·
[License: MIT](https://github.com/Effect-TS/effect/blob/main/LICENSE)

## Overview

Effect is a library — closer to a runtime and standard library — for writing
TypeScript applications where side effects, errors, and dependencies are values
rather than untracked control flow. Its central abstraction is `Effect<A, E, R>`:
a lazy description of a computation that, when run, either succeeds with `A`,
fails with a typed error `E`, or requires services `R` from its environment.
Nothing executes until an effect is handed to a runtime, which makes programs
composable and referentially transparent in the way a `Promise` is not[^1].

The project descends from the functional-programming lineage of `fp-ts`
(also by Giulio Canti) and is heavily modeled on Scala's ZIO[^2]. Where `fp-ts`
exposed raw category-theory abstractions, Effect packages the same ideas —
typed errors, dependency injection, structured concurrency, resource safety —
behind an API meant to be used by application engineers, not only FP
specialists. The defining tension is exactly that: Effect asks you to rewrite
how you express ordinary TypeScript (async/await, `try/catch`, DI containers,
`Promise.all`) in its own vocabulary, and in return gives you a compiler that
tracks which errors and dependencies each function actually has. Teams either
find that trade transformative or find the learning curve and all-or-nothing
adoption cost too high.

Effect is a monorepo. The core `effect` package now absorbs what were once
separate libraries (Schema, Stream, STM, metrics), while `@effect/*` packages
add platform bindings, CLI, SQL, RPC, and OpenTelemetry integration[^3].

## Getting Started

```sh
npm install effect@latest      # v3, the current stable line
# npm install effect@beta      # v4 beta — the `main` branch is v4 development[^4]
```

```ts
import { Effect, Console } from "effect"

class UserNotFound {
  readonly _tag = "UserNotFound"           // typed, discriminated error
  constructor(readonly id: number) {}
}

const getUser = (id: number): Effect.Effect<string, UserNotFound> =>
  id === 1
    ? Effect.succeed("Tom")
    : Effect.fail(new UserNotFound(id))

const program = Effect.gen(function* () {   // generator = do-notation
  const name = yield* getUser(1)
  yield* Console.log(`Hello ${name}`)
}).pipe(
  Effect.catchTag("UserNotFound", (e) =>    // exhaustively handled by _tag
    Console.log(`No user ${e.id}`)
  )
)

Effect.runPromise(program)                  // nothing ran until this line
```

The error channel (`UserNotFound`) is part of the type. Removing the
`catchTag` makes the failure surface in `program`'s signature, and forgetting
to handle a possible error becomes a type error rather than a runtime surprise.

## Architecture / How It Works

An `Effect` value is a plain immutable data structure describing work — not a
running task. Execution is performed by an interpreter (the **fiber runtime**)
that walks this description on lightweight green threads called **fibers**.
Fibers are the unit of concurrency: `Effect.fork` spawns one, and Effect
enforces **structured concurrency** — a forked fiber's lifetime is bound to its
scope, so when a parent completes or is interrupted, its children are
interrupted and their resources released deterministically[^5]. This is the
substance behind `Effect.all`, `race`, `timeout`, and interruption — behavior
that `Promise` cannot express because a `Promise` is already running and cannot
be cancelled.

The three type parameters map to three orthogonal concerns:

- **`A` (success)** — the value.
- **`E` (error)** — recoverable, typed failures, usually discriminated unions
  keyed by a `_tag`. Distinct from *defects* (unexpected exceptions), which
  travel a separate channel and are not meant to be caught structurally.
- **`R` (requirements)** — services the effect needs. `R` is discharged through
  **Layer** and **Context**: a `Layer` is a recipe for constructing a service
  (with its own acquisition/release and dependencies), and providing all
  required layers reduces `R` to `never`, at which point the effect can run.
  This is dependency injection encoded in the type system rather than a
  container resolved at runtime[^6].

Around this core sit **Schema** (bidirectional encode/decode with static type
inference, the parsing and validation layer), **Stream** (pull-based async
streams with backpressure), **STM** (software transactional memory for
lock-free shared state), and a **scheduler/metrics/tracing** stack that emits
OpenTelemetry spans natively. The `@effect/platform` packages abstract Node,
Bun, and browser APIs (HTTP, filesystem, workers) behind Effect services so the
same program runs across runtimes.

## Production Notes

- **Adoption is all-or-nothing at the boundary.** Effect composes beautifully
  with itself and awkwardly with everything else. You will spend real effort at
  the seams — wrapping third-party Promises with `Effect.tryPromise`, mapping
  their throws into typed errors, and deciding where `runPromise`/`runFork`
  lives. Partial adoption ("just use it in this one module") tends to leak.
- **The learning curve is the cost.** The API surface is large and the
  vocabulary (Layer, Scope, Fiber, Ref, Deferred, Semaphore, STM) is
  unfamiliar. Onboarding an engineer unfamiliar with ZIO-style effect systems
  is measured in weeks, not days. This is the single most-cited reason teams
  back out.
- **Version churn has been significant.** Effect reached a stable single-package
  2.0 relatively recently, 3.x consolidated further, and **v4 is currently in
  beta on `main`**[^4]. If you start today, pin `effect@latest` (v3) for
  production; do not build on `@beta` unless you accept breaking changes.
  Issues and PRs for the stable line target the `v3` branch.
- **Bundle size and tree-shaking.** The core is large. It is ESM-first and
  designed to tree-shake, but naive imports and older bundler configs can pull
  in more than expected; verify final bundle size for browser/edge targets.
- **Stack traces and debugging.** Because control flow runs through the fiber
  interpreter, raw stack traces are less direct than plain async/await. Effect
  invests in its own trace annotations and span data to compensate, but
  expect the debugging model to differ from what your team knows.
- **Ecosystem maturity.** Documentation and the surrounding tooling have
  improved markedly, but this is a fast-moving library with a smaller labor
  pool than mainstream TypeScript. Hiring for existing Effect experience is
  hard; you will mostly be training.

## When to Use / When Not

**Use when:**
- You want typed errors and dependency injection enforced by the compiler
  across a whole codebase, not bolted on per-module.
- You have genuinely concurrent work — cancellation, racing, resource
  lifetimes, retries with backoff — where `Promise` semantics fall short.
- You are building long-lived backend services and want built-in observability
  (OpenTelemetry), schema-driven boundaries, and a uniform error model.
- The team is willing to invest in the paradigm and standardize on it.

**Avoid when:**
- The codebase is small, mostly CRUD, or you need incremental adoption without
  committing the whole team to the model.
- Your team has no appetite for a new mental model; the curve will dominate.
- You only need one piece (validation, a Result type) — reach for a focused
  library instead of the full runtime.
- You require long-term API stability today and cannot absorb the v3→v4
  transition on the horizon.

## Alternatives

- gcanti/fp-ts — Effect's predecessor and now effectively in maintenance in
  favor of Effect; use it when you want lightweight HKT-based FP primitives
  without a runtime or fibers.
- colinhacks/zod — use when you only need standalone schema validation and
  inference, not an effect system; far smaller scope than Effect Schema.
- supermacro/neverthrow — use when all you want is a typed `Result`/`Either` for
  error handling, with no concurrency, DI, or runtime.
- ReactiveX/rxjs — use when the problem is event/stream composition over time
  and you don't need typed errors or dependency tracking in the type.
- native async/await + a DI container — use when the team values familiarity and
  the concurrency needs are modest; you give up compiler-tracked errors and
  structured concurrency.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-11-13 | `Effect-TS` monorepo begins; early `@effect-ts/*` packages, closely modeled on ZIO[^2]. |
| 2.0 | 2024-02 | Consolidation into a single `effect` package; first broadly stable public API[^1]. |
| 3.0 | 2024-04 | Further core consolidation; Schema and other modules folded toward the core. |
| v4 (beta) | 2026 | Under active development on `main`; installable as `effect@beta`. v3 remains the stable line on the `v3` branch[^4]. |

## References

[^1]: Effect documentation — introduction and the `Effect` data type. https://effect.website/docs/getting-started/the-effect-type/
[^2]: Effect FAQ / "Why Effect" — lineage from fp-ts and inspiration from Scala's ZIO. https://effect.website/docs/getting-started/why-effect/
[^3]: Effect packages overview (`effect`, `@effect/platform`, `@effect/cli`, `@effect/sql`, `@effect/rpc`, `@effect/opentelemetry`). https://effect.website/docs/
[^4]: Repository README — "Effect V4 is currently in beta. The `main` branch contains v4 development"; v3 source on the `v3` branch. https://github.com/Effect-TS/effect
[^5]: Effect documentation — fibers, forking, and structured concurrency. https://effect.website/docs/concurrency/fibers/
[^6]: Effect documentation — services, Context, and Layer for dependency injection. https://effect.website/docs/requirements-management/layers/

## Tags

typescript, javascript, effect-system, functional-programming, typed-errors, dependency-injection, structured-concurrency, schema-validation, observability, runtime-library
