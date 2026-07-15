# apple/swift-async-algorithms

> Standard-library-adjacent algorithms for `AsyncSequence` — the values-over-time operators (`merge`, `zip`, `debounce`, `combineLatest`) that Swift concurrency shipped without.

[GitHub repo](https://github.com/apple/swift-async-algorithms) ·
[License: Apache-2.0](https://github.com/apple/swift-async-algorithms/blob/main/LICENSE.txt)

## Overview

Swift Async Algorithms is a Swift Package Manager library, maintained under
the `apple` GitHub organization, that supplies algorithms operating over
`AsyncSequence`. Swift 5.5 introduced `AsyncSequence` and `for await` as
language features[^1], but shipped only a thin set of operators (`map`,
`filter`, `reduce`, etc.). This package fills the gap with the harder-to-write
combinators — the ones about *order* (`merge`, `zip`, `combineLatest`, `chain`)
and about *time* (`debounce`, `throttle`, `AsyncTimerSequence`) — plus a
back-pressured `AsyncChannel` type for hand-feeding values into a sequence.

The package's stated reason to exist is that these operators are deceptively
hard. Anyone can write `map`; correctly implementing `merge` over three async
sequences — cancelling children, propagating errors, respecting back pressure,
staying `Sendable` — is a minefield of edge cases. Centralizing that in one
tested, documented module means every Swift app gets the correct version rather
than a subtly-broken local reimplementation. It is, in spirit, the async analog
of the older `apple/swift-algorithms` package for synchronous `Sequence`.

It sits at an awkward layer, though. It is published by Apple and lives beside
the standard library conceptually, but it is *not* in the standard library and
is not bundled with the toolchain — you add it as an explicit dependency. Much
of what it does overlaps conceptually with Apple's own Combine framework, and
the two do not interoperate directly. That "official but not built-in, and
adjacent to a competing first-party framework" positioning is its defining
tension.

## Getting Started

Add it to `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/apple/swift-async-algorithms", from: "1.0.0"),
],
targets: [
    .target(name: "MyTarget", dependencies: [
        .product(name: "AsyncAlgorithms", package: "swift-async-algorithms"),
    ]),
]
```

```swift
import AsyncAlgorithms

// merge: interleave two streams into one, finishing when both finish.
for await line in merge(stdoutLines, stderrLines) {
    print(line)
}

// debounce: only emit after input goes quiet for 300ms — classic
// search-as-you-type coalescing.
for await query in searchField.textStream.debounce(for: .milliseconds(300)) {
    await runSearch(query)
}
```

## Architecture / How It Works

Everything is built on the `AsyncSequence` protocol: each algorithm is a
wrapper sequence with a bespoke `AsyncIterator` that pulls from one or more
upstream iterators. There is no scheduler or runtime; the operators are lazy and
demand-driven, advancing only when the consumer calls `next()`. This keeps the
model aligned with structured concurrency — an async `for` loop *is* the driver.

Two design threads run through the code:

- **Back pressure.** Time-based operators (`debounce`, `throttle`, `buffer`) and
  `AsyncChannel` are explicit about what happens when the producer outruns the
  consumer. `AsyncChannel.send(_:)` suspends until a consumer is ready to
  receive, giving true back pressure rather than an unbounded buffer[^2]. This is
  the sharpest behavioral contrast with Combine's `Subject`/`PassthroughSubject`,
  which drop or buffer without suspending the producer.
- **Clocks.** Time operators are generic over Swift's `Clock` protocol, so tests
  inject a `ContinuousClock`, `SuspendingClock`, or a custom fake to advance time
  deterministically instead of sleeping. This makes `debounce`/`throttle` unit-
  testable — historically a weak spot for reactive libraries.

The package also documents per-algorithm **effects** — whether an operator
throws, rethrows, or never throws, and whether its `Sendable` conformance is
unconditional or conditional on its inputs being `Sendable`[^3]. This matters
under Swift's strict concurrency checking, where a conditional `Sendable`
conformance can surface as a compile error deep inside a composed pipeline.

## Production Notes

- **Strict concurrency friction.** Under Swift 6 language mode / complete
  concurrency checking, composing these operators over non-`Sendable` element
  types produces errors that point at the operator, not your data type. Expect to
  spend time making element types and closures `Sendable` before a pipeline
  compiles. This is correct behavior, but it is the most common practical
  complaint.
- **`AsyncChannel` is not a buffer.** Because `send(_:)` suspends until a
  consumer receives, a channel with no active consumer will hang the producer's
  task. It is a synchronization primitive, not a mailbox. Teams reaching for
  "a queue I can push to and drain later" are usually looking for something else
  (an actor with an array, or `AsyncStream` with a buffering policy).
- **`AsyncStream` vs `AsyncChannel`.** `AsyncStream` ships in the Swift standard
  library / Foundation and covers the common "bridge a callback API into an async
  sequence" case with a configurable buffer. Only reach into this package's
  `AsyncChannel` when you specifically need back pressure. Don't add the whole
  dependency just for stream creation.
- **No Combine bridge.** There is no built-in adapter between `AsyncSequence` and
  Combine `Publisher`. Migrating a Combine codebase is a rewrite of operator
  semantics, not a mechanical swap; `debounce`/`throttle` timing edge cases
  differ.
- **Toolchain coupling.** The package deliberately tracks recent Swift toolchains
  and states that requiring a newer Swift release only warrants a minor version
  bump[^4]. Pinning an old Swift compiler can block upgrades.
- **Maturity.** At ~3.7k stars with an Apache-2.0 license and commits into 2026,
  it is actively maintained but low-churn — the API surface is largely settled
  post-1.0, and issue volume reflects a stable, narrowly-scoped library rather
  than an active development frontier.

## When to Use / When Not

**Use when:**
- You already write Swift concurrency (`async`/`await`, `AsyncSequence`) and need
  `merge`, `zip`, `combineLatest`, `debounce`, or `throttle`.
- You want deterministic, `Clock`-injectable time-based operators for testing.
- You need genuine back pressure between an async producer and consumer
  (`AsyncChannel`).

**Avoid when:**
- You only need to bridge a delegate/callback API into an async sequence — the
  standard library's `AsyncStream` already does that without a dependency.
- Your codebase is committed to Combine and its scheduler/back-pressure model;
  mixing the two adds two mental models with no bridge.
- You target Swift toolchains too old to adopt the current release, or you cannot
  take on strict-concurrency migration work.

## Alternatives

- apple/swift-algorithms — the synchronous-`Sequence` sibling; use it when your
  data is already in memory and not time-based.
- Combine (Apple, closed-source, bundled with Apple SDKs) — use it when you're on
  Apple platforms only and want an integrated reactive framework with schedulers.
- ReactiveX/RxSwift — use it when you need cross-team Rx operator parity or are
  porting Rx code from another platform.
- pointfreeco/swift-dependencies + AsyncStream — use plain `AsyncStream` when you
  only need stream creation and buffering, not the combining/time operators.
- CombineCommunity/CombineExt — use it when you're staying in Combine but missing
  a specific operator.

## History

| Version | Date | Notes |
|---------|------|-------|
| Repo created | 2022-01-10 | Public development begins under `apple` org[^5]. |
| 0.0.x–0.1.x | 2022 | Pre-1.0 previews; API in flux, developed openly on Swift Forums. |
| 1.0.0 | 2023-09 | First source-stable release; SemVer guarantees begin[^4]. |
| Maintenance | 2024–2026 | Ongoing fixes and Swift 6 / strict-concurrency alignment; last push 2026-06-29[^5]. |

## References

[^1]: Swift Evolution SE-0298, "Async/Await: Sequences" — introduced `AsyncSequence` in Swift 5.5. https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md
[^2]: `AsyncChannel` guide (back-pressure sending semantics). https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md
[^3]: Effects guide (throwing / Sendability effects per algorithm). https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Effects.md
[^4]: README, "Source Stability" — SemVer, source-breaking changes only in major versions, Swift-toolchain bumps are minor. https://github.com/apple/swift-async-algorithms#source-stability
[^5]: GitHub repository metadata, apple/swift-async-algorithms (created 2022-01-10, last push 2026-06-29). https://github.com/apple/swift-async-algorithms

## Tags

swift, async-await, concurrency, asyncsequence, reactive, back-pressure, apple, spm, structured-concurrency, streams
