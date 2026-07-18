# clojure/core.async

> CSP-style channels and go blocks for Clojure and ClojureScript — concurrency
> as a library, not a language feature.

[GitHub repo](https://github.com/clojure/core.async) ·
[API docs](https://clojure.github.io/core.async/) ·
[License: EPL-1.0](https://github.com/clojure/core.async#license)

## Overview

core.async brings Communicating Sequential Processes — channels plus
lightweight processes that park on them — to Clojure and ClojureScript.
Announced by Rich Hickey in June 2013[^1], it deliberately follows Go's
channel model rather than promises or actors: the rationale argues channels
decouple producers from consumers better than callback chains or shared
state[^2]. It is maintained by the Clojure core team, which shapes how the
project operates: no pull requests, no GitHub issues (JIRA plus a signed
Contributor Agreement), and a MAJOR.MINOR.COMMITS version scheme that
explicitly does not follow semver[^3].

The defining tension is "library, not language." The `go` macro rewrites your
code into a state machine at compile time, so Clojure gets goroutine-like
semantics without runtime support — but the illusion leaks. Parking ops
(`<!`, `>!`) only work in the literal body of a `go` block, not in functions
called from it; blocking I/O inside `go` starves a small shared pool; and
ClojureScript gets the same syntax with no blocking variants at all. Go the
language pays for goroutines in its runtime; core.async makes you pay at the
edges of a macro.

At ~2,000 stars the repo looks small for its influence. That undercounts it:
distribution is via Maven Central, the org accepts no PRs (no star flywheel),
and it has been the Clojure ecosystem's default async vocabulary for a decade.
Activity is real — 2026 push cadence, virtual-thread integration, and a new
`core.async.flow` layer in alpha[^3].

## Getting Started

```clojure
;; deps.edn
{:deps {org.clojure/core.async {:mvn/version "1.9.865"}}}
```

```clojure
(require '[clojure.core.async :as a :refer [chan go <! >! <!! thread]])

(def c (chan 10))                       ; buffered channel, capacity 10

(go (>! c (str "result-" (rand-int 100))))  ; parking put, runs on pool
(<!! c)                                 ; blocking take, ordinary thread
;; => "result-42"

(thread (a/>!! c :from-a-real-thread))  ; real thread for blocking work
(go (println (a/alts! [c (a/timeout 1000)])))  ; select with timeout
```

Never use `<!!`/`>!!` (blocking) inside a `go` block, and never use `<!`/`>!`
(parking) outside one — the former deadlocks the pool, the latter throws.

## Architecture / How It Works

The core trick is the **`go` macro**: it runs the body through an analyzer
(`tools.analyzer.jvm` on the JVM), converts it to SSA form, and emits a state
machine whose states are the segments between parking points. Parking
registers a callback on the channel and returns the thread to the pool; the
machine resumes when the operation completes, so no threads block while a
`go` block waits. Tim Baldridge's talks remain the best internals
walkthrough[^4].

**Channels** are lock-based queues with three buffer policies — fixed
(backpressure by parking), `dropping-buffer`, `sliding-buffer` — plus
unbuffered rendezvous channels. At most 1024 pending puts or takes are
allowed; exceeding that throws, a guard against unbounded callback buildup.
Channels accept transducers (`(chan 10 (map parse))`, applied on the
producer's thread). `nil` cannot be conveyed; taking `nil` signals closed.

**Dispatch** historically ran on a fixed pool of 8 threads (the source of the
classic starvation footgun). Recent releases replaced the fixed pool with a
pluggable `clojure.core.async.executor-factory` and use JVM virtual threads
where available for `go` and `io-thread`[^3] — a significant quiet
modernization of the runtime story.

**core.async.flow**, alpha since 2025, is a higher layer: processes and their
channel connections are declared as data, and flow assembles, starts, pauses,
and monitors the graph[^3] — aimed at the long-standing critique that raw
channel topologies are invisible and hard to operate. APIs are still moving.

On **ClojureScript**, the same surface compiles to a state machine on the JS
event loop. `<!!`, `>!!`, and `thread` do not exist there; cross-platform
code must stick to the parking subset.

## Production Notes

- **Pool starvation is the canonical incident.** Any blocking call — JDBC,
  HTTP, `Thread/sleep`, a contended lock — inside a `go` block occupies a
  dispatch thread; under the historical 8-thread pool, eight such calls froze
  every `go` block in the process. Use `thread` or `io-thread` for blocking
  work; set the `clojure.core.async.go-checking` property in dev to assert
  against blocking ops in `go`[^3].
- **The fn-boundary rule bites during refactoring.** Extracting code
  containing `<!` into a helper function breaks it — the macro cannot see
  across function boundaries. Ordinary Clojure decomposition is hazardous
  inside `go` bodies; this is the most common newcomer trap.
- **Failure signaling is quiet.** A throw inside `go` does not surface on
  the result channel — it just closes. Puts to a closed channel return false
  rather than throwing; takes return `nil`. Loops that don't check `nil`
  spin, code ignoring put results silently drops data, and errors need
  explicit try/catch conveying them as values.
- **Versioning requires attention.** MAJOR.MINOR.COMMITS is not semver;
  1.6.673 raised the Clojure minimum to 1.10, and flow alphas ship on the
  same artifact with higher numbers than stable (1.10.x-alpha vs
  1.9.865)[^3]. Pin exact versions.
- **Virtual threads change the calculus.** On JDK 21+, `io-thread` and the
  executor integration use virtual threads[^3]; for JVM-only code, virtual
  threads with blocking ops (`<!!`) are now a reasonable alternative to `go`
  state machines — the compile-time transform matters most where virtual
  threads don't exist (ClojureScript).
- **Governance is slow by design.** No PRs, JIRA-only issues, releases years
  apart on the stable line. Stability is excellent — changes move to new
  names rather than breaking old ones[^3] — but edge-case fixes are slow.

## When to Use / When Not

**Use when:**
- You need coordination between concurrent processes — fan-in, fan-out,
  select (`alts!`), pipelines — not just "run this async."
- You need one async model across JVM Clojure and ClojureScript.
- You want backpressure by construction (bounded buffers, parking puts).
- It is the async lingua franca other Clojure libraries assume.

**Avoid when:**
- You just need async values with success/failure — promise-shaped tools
  compose more simply and keep error handling explicit.
- JVM-only on JDK 21+: virtual threads plus blocking queues or plain
  `<!!`-style code may beat `go` machinery in debuggability.
- Your team is new to Clojure — the footgun list above (fn boundaries,
  starvation, swallowed exceptions) has real onboarding cost.
- You need streaming with rich operators (windowing, joins) — channels are
  low-level primitives, not a streams API.

## Alternatives

- clj-commons/manifold — deferreds + streams that interop with promises and
  channels; use it when you want async values and the Aleph ecosystem.
- funcool/promesa — CompletableFuture-based promises for CLJ/CLJS; use it
  when your async is request/response-shaped, not process-shaped.
- leonoel/missionary — functional effect and dataflow system; use it for
  supervised, cancellable composition under a stricter model.
- golang/go — if CSP is the reason you're here, Go has it in the runtime
  with none of the macro edge cases.
- Java virtual threads (JDK 21+) — plain threads plus `BlockingQueue`; use
  when JVM-only and simplicity wins.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2013-06 | Announced; go macro, channels, alts![^1] |
| 1.6.673 | 2022-10 | Requires Clojure 1.10; go compilation perf[^3] |
| 1.7.701 | 2024-12 | Direct delivery for blocking-op completions[^3] |
| 1.8.730 | 2025-04 | `io-thread`, pluggable `executor-factory`; fixed `pool-size` property removed[^3] |
| 1.9.808-alpha1 | 2025-04 | First core.async.flow alpha[^3] |
| 1.9.865 | 2026-03 | Virtual threads in `io-thread`; `alts!!` caller dispatch; datafy for channels[^3] |
| 1.10.870-alpha2 | 2026-04 | flow line continues in alpha; futurize exception fix[^3] |

## References

[^1]: Rich Hickey, "Clojure core.async Channels" — 2013-06-28. https://clojure.org/news/2013/06/28/clojure-clore-async-channels
[^2]: core.async Rationale. https://clojure.github.io/core.async/rationale.html
[^3]: core.async README — releases, changelog, contribution policy. https://github.com/clojure/core.async#changelog
[^4]: Tim Baldridge, go macro internals (Clojure/conj 2013). https://www.youtube.com/watch?v=R3PZMIwXN_g

## Tags

clojure, clojurescript, csp, channels, concurrency, async, go-macro, jvm,
backpressure, dataflow
