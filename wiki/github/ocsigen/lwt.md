# ocsigen/lwt

> OCaml's incumbent concurrency library — cooperative promises and non-blocking
> I/O in a single thread, predating async/await by a decade.

[GitHub repo](https://github.com/ocsigen/lwt) ·
[Official website](https://ocsigen.org/lwt) ·
[License: MIT](https://github.com/ocsigen/lwt/blob/master/LICENSE.md)

## Overview

Lwt ("lightweight threads", a historical name the maintainers now regret —
`'a Lwt.t` is a promise, not a thread[^1]) is a concurrent programming library
for OCaml. It provides one core abstraction: a promise that resolves in the
future, composed monadically with `Lwt.bind`. All user code runs cooperatively
in a single thread, so there is no locking or preemption to reason about; I/O
runs in parallel underneath via non-blocking file descriptors and a small
worker-thread pool. Lwt grew out of Jérôme Vouillon's work in the Ocsigen web
framework project and was described in a 2008 ML Workshop paper[^2], making it
one of the earliest widely-deployed promise systems in any language.

For most of OCaml's history the ecosystem was split between two incompatible
concurrency monads: Lwt and Jane Street's Async. Lwt won the broader community
— MirageOS unikernels, cohttp, js_of_ocaml frontends, and most of the opam
ecosystem target it. The defining tension today is different: OCaml 5 (2022)
shipped effect handlers and multicore, enabling direct-style concurrency
libraries like Eio that need no monad at all. Lwt is not going away — it is
actively maintained (pushed within days as of mid-2026) and remains the safe
default for the existing ecosystem — but it is now the incumbent rather than
the frontier, and its single-threaded model does not exploit multiple cores.

## Getting Started

```bash
# libev is the recommended event loop backend (libev-dev / libev-devel)
opam install conf-libev lwt
```

```ocaml
(* dune: (executable (name main) (libraries lwt lwt.unix)) *)
open Lwt.Syntax

let fetch_delayed name secs =
  let* () = Lwt_unix.sleep secs in
  Lwt_io.printlf "%s done after %.1fs" name secs

let () =
  Lwt_main.run
    (Lwt.join
       [ fetch_delayed "a" 1.0; fetch_delayed "b" 0.5 ])
    (* both run concurrently; total wall time ~1.0s, not 1.5s *)
```

`Lwt.Syntax` provides `let*` binding operators; the `lwt_ppx` package provides
the older `let%lwt` syntax. Both desugar to `Lwt.bind`.

## Architecture / How It Works

- **Promises, three states.** `'a Lwt.t` is pending, fulfilled, or rejected.
  `Lwt.bind` attaches a callback; `Lwt.return` wraps a value. Combinators:
  `Lwt.join` (wait for all), `Lwt.pick`/`Lwt.choose` (race), `Lwt.both`.
- **Callbacks run immediately, not on a microtask queue.** Binding on an
  already-resolved promise executes the callback synchronously, unlike
  JavaScript promises. This makes resolved-path code fast but means callback
  ordering is subtler and deep synchronous chains can grow the stack;
  `Lwt.pause` explicitly yields to the scheduler.
- **The scheduler is an event loop.** `Lwt_main.run` drives an "engine"
  (`Lwt_engine`): libev when `conf-libev` is installed (epoll/kqueue under the
  hood), otherwise a `select`-based fallback[^3]. Windows uses select.
- **Syscalls that cannot be made non-blocking** — `getaddrinfo`, most file I/O
  on Linux — are detached to a pool of worker system threads by `Lwt_unix`, so
  they resolve in parallel without blocking the main thread. `Lwt_preemptive`
  exposes this detachment for arbitrary user code.
- **Layering.** The core `Lwt` module is pure OCaml with no Unix dependency,
  which is why it compiles cleanly to JavaScript via js_of_ocaml (the browser
  event loop replaces the engine) and runs inside MirageOS unikernels. The
  Unix binding (`Lwt_unix`, `Lwt_io`, `Lwt_process`) is a separate sublibrary
  wrapping nearly the entire Unix API.
- **Error channel.** A rejected promise carries an exception. Exceptions
  raised inside a callback are caught and become rejections, but code before
  the first bind raises synchronously — the reason `Lwt.catch` (not `try`)
  is the required error-handling form across await points.

## Production Notes

- **Cooperative starvation is the classic footgun.** A long pure computation
  (parsing a large file, a tight loop) blocks every other promise, timers
  included. There is no preemption; you must insert `Lwt.pause` manually or
  detach to `Lwt_preemptive`. Latency-sensitive services need auditing for
  this.
- **Cancellation is the weakest part of the design.** `Lwt.cancel` propagates
  backwards through the promise graph with results that are hard to predict
  once promises are shared; the manual itself steers users toward explicit
  `Lwt.pick`-with-timeout patterns instead, and the API is discouraged for
  new code[^4]. Do not build correctness-critical logic on cancellation.
- **The monad colors your code.** Every function touching I/O returns
  `'a Lwt.t`, and every caller must bind. Mixing Lwt and Async in one program
  is effectively unsupported; library authors historically shipped `foo-lwt`
  and `foo-async` variants. Choose your monad once, at project start.
- **Backtraces are poor.** Exceptions crossing binds lose stack context;
  `lwt_ppx` does some backtrace re-raising, but debugging rejected promises
  is measurably worse than debugging synchronous OCaml.
- **Engine choice matters at scale.** The select fallback caps at
  `FD_SETSIZE` (typically 1024) descriptors; install `conf-libev` for any
  real server workload.
- **OCaml 5 status.** Lwt runs on OCaml 5 but stays single-domain; the
  scheduler is not parallel. The companion `lwt_domain` offloads CPU-bound
  work to other domains via Domainslib, but if you want parallelism-native
  concurrency the community direction is Eio or Miou. Bindings between Lwt
  and Eio exist (lwt_eio) to support incremental migration.
- **`Lwt_io` channels are buffered** — forgetting to flush (or to use the
  `with_connection`/`with_file` bracket functions) is a common source of
  "my output never arrived" bugs.

## When to Use / When Not

**Use when:**
- You are writing OCaml network services with the existing ecosystem —
  cohttp, opium, dream (which uses Lwt), resto, irmin: most of it speaks Lwt.
- You target MirageOS unikernels or js_of_ocaml, where Lwt is the assumed
  concurrency layer.
- You need mature, battle-tested concurrency on OCaml 4.x, where effects are
  unavailable.

**Avoid when:**
- You are starting a fresh OCaml 5 project with no legacy dependencies —
  direct-style Eio avoids the monad tax and can use multiple cores.
- Your workload is CPU-bound parallelism, not I/O concurrency — Lwt's
  single-threaded model gives you nothing; use Domainslib.
- Your team is deep in the Jane Street Core/Async stack — mixing monads is
  not practical.

## Alternatives

- ocaml-multicore/eio — use instead for new OCaml 5 code; effects-based
  direct style, multicore-capable, no monad.
- janestreet/async — use instead if you are already committed to the Jane
  Street Core ecosystem; same monadic shape, incompatible ecosystem.
- ocaml/domainslib — use instead when the problem is CPU-bound data
  parallelism across cores rather than I/O concurrency.
- robur-coop/miou — use instead if you want a minimal effects-based scheduler
  with stricter resource-handling opinions than Eio.
- riot-ml/riot — use instead for Erlang-style actor/process semantics on
  OCaml 5.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2008 | Vouillon's ML Workshop paper describes the design[^2]. |
| 2.x | 2010s | Long stable era; camlp4 `lwt ... >>` syntax. |
| 3.0 | 2017 | Modernization under new maintainer Anton Bachin; build moved to jbuilder/dune era, deprecations begin[^5]. |
| 4.0 | 2018 | PPX split into separate `lwt_ppx` package; camlp4 syntax retired. |
| 5.0 | 2019 | Breaking cleanup of callback semantics and deprecated APIs. |
| 5.6+ | 2022– | OCaml 5 compatibility; `lwt_domain` companion for multicore offload. |
| 5.9.x | 2025–2026 | Current line; maintenance releases, active as of 2026-07[^3]. |

## References

[^1]: Lwt README — "`'a Lwt.t` is a promise, and has nothing to do with system or preemptive threads." https://github.com/ocsigen/lwt#readme
[^2]: Jérôme Vouillon, "Lwt: a cooperative thread library", ACM SIGPLAN Workshop on ML, 2008. https://www.irif.fr/~vouillon/publi/lwt.pdf
[^3]: Lwt manual and API documentation. https://ocsigen.org/lwt/
[^4]: Lwt manual, `Lwt.cancel` documentation and cancellation caveats. https://ocsigen.org/lwt/latest/api/Lwt
[^5]: Lwt release announcements, discuss.ocaml.org. https://discuss.ocaml.org/tag/lwt

## Tags

ocaml, concurrency, promises, async-io, event-loop, cooperative-scheduling, libev, mirageos, functional-programming, library
