# erlang/otp

> The Erlang language, the BEAM virtual machine, and the OTP libraries — the runtime behind soft real-time, fault-tolerant, always-on systems.

[GitHub repo](https://github.com/erlang/otp) ·
[Official website](https://www.erlang.org) ·
[License: Apache-2.0](https://github.com/erlang/otp/blob/master/LICENSE.txt)

## Overview

Erlang is a functional, concurrency-oriented language created at Ericsson in the late 1980s (Joe Armstrong, Robert Virding, Mike Williams) to run telephone switches that must never stop[^1]. It was open-sourced in 1998. This repository is the canonical implementation: the compiler, the BEAM virtual machine (the runtime), and OTP — a set of libraries and, more importantly, a set of design principles for building supervised, distributed, hot-upgradable systems. In practice "Erlang" and "OTP" are inseparable; you ship OTP applications, not bare Erlang.

The defining idea is the **process**: a lightweight, isolated unit of execution with its own heap that shares nothing and communicates only by asynchronous message passing. A single BEAM node routinely runs hundreds of thousands to millions of processes, preemptively scheduled across one run queue per CPU core. Because heaps are per-process, garbage collection is per-process — there is no global stop-the-world pause — which is why the platform delivers predictable tail latencies under load rather than raw throughput.

The second defining idea is **"let it crash"**: instead of defensive error handling, a process that hits an unexpected state dies, and a *supervisor* restarts it into a known-good state. Fault tolerance is a structural property of the supervision tree, not something bolted on. The tradeoff is that Erlang is a niche, opinionated ecosystem: the syntax is unusual (Prolog-derived), the standard library is idiosyncratic, and the language is deliberately bad at the things it was never meant for — CPU-bound numeric work and single-threaded raw speed.

## Getting Started

```sh
# Debian/Ubuntu
apt-get install erlang
# macOS
brew install erlang
# Version-managed builds (recommended for multiple OTP versions)
# https://github.com/kerl/kerl  or  asdf plugin
```

```erlang
-module(hello).
-export([world/0]).

world() -> io:format("Hello, world~n").
```

```sh
$ erl
Eshell V15.0  (abort with ^G)
1> c(hello).
{ok,hello}
2> hello:world().
Hello, world
ok
```

A minimal supervised server using the `gen_server` behaviour:

```erlang
-module(counter).
-behaviour(gen_server).
-export([start_link/0, bump/0, init/1, handle_call/3, handle_cast/2]).

start_link() -> gen_server:start_link({local, ?MODULE}, ?MODULE, 0, []).
bump()       -> gen_server:call(?MODULE, bump).

init(N)                     -> {ok, N}.
handle_call(bump, _From, N) -> {reply, N + 1, N + 1}.
handle_cast(_Msg, N)        -> {noreply, N}.
```

## Architecture / How It Works

**BEAM** is the register-based virtual machine that executes compiled Erlang bytecode. Its scheduling and memory model — not the language surface — is what makes the platform distinct:

- **Processes** are green threads managed entirely by BEAM, not OS threads. Spawning one costs a few microseconds and a few hundred bytes. They are preemptively scheduled by *reduction counting*: each process runs a fixed budget of reductions (roughly, function calls) before yielding, so a single runaway process cannot starve the others.
- **Schedulers** — one per logical core by default — each own a run queue and steal work from each other. I/O and timers are handled by separate poll/async threads.
- **Message passing** copies data between process heaps (shared-nothing), except large binaries (>64 bytes), which live in a reference-counted off-heap area and are passed by reference.
- **Per-process GC** means collection pauses are bounded by one small heap, not the whole VM.

**OTP behaviours** are the reusable skeletons that encode correct concurrency patterns so applications don't re-implement them: `gen_server` (request/response server), `gen_statem` (state machine), `supervisor` (restart strategy), and `application` (start/stop unit). A supervision tree wires these into a hierarchy where each supervisor defines how and how often to restart failing children.

**Hot code loading** lets BEAM hold two versions of a module simultaneously; a running system can be upgraded without dropping connections. This is real and used in telecom, but most modern shops do rolling restarts instead because release upgrades (`relup`/`appup`) are difficult to author and test.

**Distribution** is built in: nodes connect into a mesh and send messages to remote processes with the same `!` operator used locally. This transparency is elegant and also a footgun — see Production Notes. Since OTP 24 (2021), BEAM includes a **JIT** (BeamAsm) that compiles bytecode to native machine code at load time, meaningfully improving throughput over the old threaded interpreter[^2].

## Production Notes

**Distributed Erlang does not scale to large clusters.** The default transport builds a fully-connected mesh with all-to-all heartbeats, which practically caps a single cluster around 50–100 nodes. Larger topologies need partitioning, a custom `dist` layer, or libraries like Partisan. The classic distribution auth is a shared **cookie** — a plaintext atom — and the node discovery daemon (`epmd`) plus the distribution port are unauthenticated by default; production clusters must run distribution over TLS and firewall these ports.

**Mnesia is convenient and treacherous.** The built-in distributed database is great for small configuration/session state, but it has no automatic conflict resolution on network partition (you get split-brain and must merge manually), and historically carried size limits on disc-based tables. Reach for PostgreSQL, or an external store, once data matters.

**NIFs and long work block schedulers.** Native Implemented Functions (C code called from Erlang) run *on* a scheduler thread. A NIF that runs longer than ~1 ms, or crashes, degrades or takes down the whole VM. Long native work must use **dirty schedulers** or be broken into chunks. The same discipline applies to any tight, reduction-cheap loop.

**It is not for number crunching.** Floats are boxed, there is no SIMD story, and per-process heaps make large shared numeric arrays awkward. CPU-bound math belongs in a NIF, a port program, or another language entirely.

**Strings are a historical wart.** Plain strings are lists of integer code points — memory-heavy and slow. Idiomatic code uses **binaries** (and `iodata`) for all real text and I/O; treat list-strings as a legacy convenience only.

**Introspection is a genuine strength.** `observer`, `recon`, and remote shells let you attach to a live node and inspect any process, its message queue, and memory. Watch for **large message queues** (a slow consumer is the most common outage cause) and **atom table exhaustion** (atoms are never garbage-collected — never create them from untrusted input).

## When to Use / When Not

**Use when:**
- You need high availability and graceful degradation: messaging, telecom, chat, payment routing, IoT/MQTT brokers, ad serving.
- You are holding many concurrent, mostly-idle, stateful connections (millions of sockets on one box).
- Predictable tail latency matters more than peak single-request speed.
- You want supervision, live introspection, and rolling upgrades as first-class runtime features.

**Avoid when:**
- The workload is CPU-bound numeric or data-crunching (use Rust, C++, Julia, or a NIF boundary).
- You need a large mainstream library ecosystem and hiring pool — both are comparatively small.
- The task is a short script or CLI tool; BEAM startup and the OTP model are overkill.
- Your team has no appetite for functional programming and the actor model's mental shift.

## Alternatives

- elixir-lang/elixir — same BEAM VM and OTP; modern Ruby-ish syntax, macros, and far better tooling (Mix, Hex). Use instead when you want everything Erlang offers with better ergonomics and docs.
- gleam-lang/gleam — statically typed language on BEAM. Use when you want compile-time type safety and still want OTP concurrency.
- golang/go — CSP goroutines and a huge ecosystem. Use when you want cheap concurrency plus mainstream libraries and hiring, and can forgo supervision/hot-reload/distribution-as-a-feature.
- akka/akka — the actor model on the JVM. Use when you are committed to the JVM ecosystem and need actors there.
- Pony — actor language with a capabilities-based type system and no data races. Use when you want actor safety guarantees enforced by the compiler.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 1986 | Developed internally at Ericsson for telecom switches[^1]. |
| — | 1998 | Open-sourced by Ericsson. |
| OTP 18 | 2015-06 | License changed from Erlang Public License to Apache-2.0[^3]. |
| OTP 21 | 2018-06 | New logger, `ETS` improvements, dirty-scheduler maturation. |
| OTP 24 | 2021-05 | BeamAsm JIT compiler enabled by default[^2]. |
| OTP 26 | 2023-05 | Improved error messages, maps performance work. |
| OTP 27 | 2024-05 | Built-in `json` module, `-doc`/`-moduledoc` attributes, triple-quoted strings, sigils[^4]. |
| OTP 28 | 2025-05 | Continued language/runtime refinement (yearly major cadence). |

## References

[^1]: Joe Armstrong, "A History of Erlang" — HOPL III, 2007. https://dl.acm.org/doi/10.1145/1238844.1238850
[^2]: Erlang/OTP 24 release announcement (BeamAsm JIT). https://www.erlang.org/news/148
[^3]: Erlang/OTP 18.0 release — license change to Apache 2.0. https://www.erlang.org/news/85
[^4]: Erlang/OTP 27 highlights. https://www.erlang.org/news/170

## Tags

erlang, otp, beam, actor-model, concurrency, fault-tolerance, distributed-systems, soft-real-time, virtual-machine, functional-programming, supervision-trees
