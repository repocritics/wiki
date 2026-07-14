# elixir-lang/elixir

> A dynamic, functional language on the Erlang VM — built for concurrency, fault tolerance, and long-running distributed systems.

[GitHub repo](https://github.com/elixir-lang/elixir) ·
[Official website](https://elixir-lang.org/) ·
[License: Apache-2.0](https://github.com/elixir-lang/elixir/blob/main/LICENSE)

## Overview

Elixir is a functional, dynamically typed language created by José Valim, first released in 2012 with a stable 1.0 in September 2014[^1]. It compiles to bytecode for the BEAM — the Erlang virtual machine — and reuses Erlang/OTP's runtime, standard library, and battle-tested concurrency primitives wholesale. Elixir's contribution is not a new runtime but a modern surface over an old, proven one: Ruby-adjacent syntax, a macro system, first-class tooling (Mix, Hex, ExUnit), and a documentation culture, all sitting on 25+ years of telecom-grade Erlang engineering.

The defining trait is inherited from the BEAM: **lightweight processes and supervision**. Concurrency is not threads or async/await but millions of cheap, isolated, share-nothing processes that communicate by message passing and are preemptively scheduled. Failures are contained per-process and recovered by supervision trees under the "let it crash" philosophy — you write the happy path and let a supervisor restart a crashed subtree into a known-good state, rather than defensively handling every error inline.

The central tension is that this model is excellent for I/O-bound, highly concurrent, long-lived systems (web backends, messaging, IoT fleets, real-time features) and mediocre for CPU-bound number crunching. The BEAM trades raw single-threaded throughput for scheduling fairness and fault isolation. Heavy computation is pushed to NIFs (native C/Rust), ports, or the Nx numerical stack — each with its own caveats.

## Getting Started

```bash
# Requires Erlang/OTP. Recommended installer bundles both:
curl -fsSO https://elixir-lang.org/install.sh
sh install.sh elixir@latest otp@latest
# or via a version manager: asdf, mise, Homebrew (`brew install elixir`)
```

```elixir
# mix.exs — new project via `mix new my_app`
# lib/my_app.ex
defmodule MyApp do
  # Pattern matching in function heads + guards
  def classify(n) when n < 0, do: :negative
  def classify(0), do: :zero
  def classify(_n), do: :positive
end

# The pipe operator threads data left-to-right
result =
  1..100
  |> Enum.filter(&(rem(&1, 3) == 0))
  |> Enum.map(&(&1 * &1))
  |> Enum.sum()
```

```elixir
# Spawn a supervised, isolated process (GenServer)
defmodule Counter do
  use GenServer
  def start_link(n), do: GenServer.start_link(__MODULE__, n, name: __MODULE__)
  def init(n), do: {:ok, n}
  def bump, do: GenServer.call(__MODULE__, :bump)
  def handle_call(:bump, _from, n), do: {:reply, n + 1, n + 1}
end
```

## Architecture / How It Works

Elixir source is compiled by the Elixir compiler (itself written mostly in Elixir, bootstrapped from Erlang) into `.beam` files — the same bytecode format Erlang produces. At runtime there is no distinction: Elixir and Erlang modules call each other with zero overhead, and the entire Erlang/OTP standard library (`:crypto`, `:gen_tcp`, `:mnesia`, `:ssl`) is directly usable.

Key layers:

- **The BEAM.** A preemptive, per-process-garbage-collected VM. Each process has its own heap; there is no shared mutable memory, so GC never stops the world — it collects one small process heap at a time. The scheduler gives every process a reduction budget and forcibly yields it, keeping the system soft-real-time even under load.
- **OTP behaviours.** `GenServer` (stateful server), `Supervisor` (restart strategies), `Task`, `Agent`, and `Application` are Erlang design patterns exposed as Elixir modules. Production systems are trees of supervised processes, not call stacks.
- **Message passing.** Processes communicate by copying messages between heaps (share-nothing). Large binaries (>64 bytes) are the exception — they live in a shared, reference-counted off-heap space.
- **Macros / metaprogramming.** Elixir code is represented as data (the quoted AST); macros run at compile time and receive/emit that AST. Much of the language — `if`, `unless`, `defmodule`, test assertions — is defined as macros in Elixir itself. This powers DSLs like Ecto queries and Phoenix routers.
- **Mix + Hex.** `mix` is the build tool and task runner; `hex.pm` is the package registry. Dependencies compile from source into your project.
- **Set-theoretic types.** Since 1.17 (2024) Elixir has been incrementally adding a gradual, set-theoretic type system that infers types from patterns and flags some errors at compile time, without type annotations[^2]. It is still being rolled out and does not yet cover the whole language.

## Production Notes

**CPU-bound work is the weak spot.** The BEAM is optimized for concurrency and fairness, not numeric throughput. Tight loops over large data are slow relative to C/Go/Rust. The escape hatches — NIFs written in C or via Rustler (Rust) — run outside the scheduler: a long-running or crashing NIF blocks a scheduler thread or takes down the entire VM. Prefer dirty schedulers for long NIFs, or offload to `Nx`/ports.

**Clustering has a ceiling.** Distributed Erlang gives you transparent inter-node messaging out of the box, but the default topology is a full mesh with all-to-all heartbeats. This works well up to roughly dozens of nodes; larger clusters need `libcluster` for discovery plus a non-mesh layer (e.g. Partisan) to avoid the connection explosion. Distribution is also not secure by default — the cookie-based auth assumes a trusted network.

**Deployment is via releases, not hot upgrades.** Erlang's famous hot-code-swapping exists but is fragile and rarely used in practice; almost everyone deploys immutable `mix release` artifacts (available since 1.9) and does rolling restarts. Releases bundle the ERTS, so the target host needs no Erlang installed — but the build host's OS/arch must match the target unless you cross-compile.

**Interop gotchas.** Erlang uses charlists (lists of integers) where Elixir uses UTF-8 binaries for strings; passing the wrong one across the boundary is a classic bug. Atoms are not garbage-collected — dynamically creating atoms from untrusted input (`String.to_atom/1` on user data) is a memory-exhaustion vector; use `String.to_existing_atom/1`.

**Observability is a genuine strength.** The VM exposes live process counts, message queue lengths, memory, and reductions; `:observer`, `recon`, and telemetry hooks let you inspect a running production node without a redeploy. This is the flip side of the concurrency model — the runtime knows a lot about itself.

**Ecosystem size.** Smaller than JavaScript/Python/JVM. Phoenix (web) and Ecto (database) are mature and dominant; outside the web niche you will sometimes wrap an Erlang library or write your own. The hiring pool is smaller, though retention tends to be high.

## When to Use / When Not

**Use when:**
- You are building highly concurrent, I/O-bound, long-running services (chat, presence, pub/sub, real-time dashboards, IoT device fleets).
- Fault tolerance and graceful degradation matter more than raw single-request latency.
- You want soft-real-time behavior with predictable tail latencies under load (no stop-the-world GC).
- Web apps where Phoenix LiveView can replace a separate SPA frontend.

**Avoid when:**
- Your workload is CPU/number-crunching heavy (ML training, video encoding, simulation) — you'll live in NIFs and lose the language's advantages.
- You need a large hiring pool and a library for everything off the shelf.
- You want static typing guarantees today across the whole codebase (the type system is still maturing).
- The problem is a simple script or CRUD app where the concurrency model is overkill.

## Alternatives

- erlang/otp — the platform underneath Elixir; use directly when you want the raw runtime, prefer Prolog-derived syntax, or need zero Elixir tooling dependency.
- gleam-lang/gleam — a statically typed, ML-flavored language on the same BEAM; use when you want compile-time type safety and BEAM concurrency together.
- golang/go — CSP-style concurrency with static types and trivial single-binary deploys; use when you want simpler ops and don't need supervision-tree fault tolerance.
- akka / Pekko (JVM) — the actor model on the JVM; use when you're committed to the JVM ecosystem and want actors there.
- nodejs/node — event-loop async for web I/O; use when the team is JS-native and workloads are moderate-concurrency rather than massively parallel.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-09 | First stable release[^1]. |
| 1.6 | 2018-01 | Built-in code formatter (`mix format`). |
| 1.9 | 2019-06 | `mix release` — self-contained deployment artifacts. |
| 1.12 | 2021-05 | Scripting improvements, `Mix.install/2` for standalone scripts. |
| 1.15 | 2023-06 | Faster compilation, cleaner compiler warnings. |
| 1.17 | 2024-06 | Set-theoretic type inference introduced; `Duration`; OTP 27 support[^2]. |
| 1.18 | 2024-12 | Type checking extended across function boundaries; built-in LSP work; parameterized ExUnit tests[^3]. |

## References

[^1]: José Valim, "Elixir v1.0 released" — 2014-09-18. https://elixir-lang.org/blog/2014/09/18/elixir-v1-0-0-released/
[^2]: "Elixir v1.17 released" — set-theoretic types and type inference. https://elixir-lang.org/blog/2024/06/12/elixir-v1-17-0-released/
[^3]: "Elixir v1.18 released" — type checking and language server. https://elixir-lang.org/blog/2024/12/19/elixir-v1-18-0-released/

## Tags

elixir, erlang, beam, functional-programming, concurrency, actor-model, otp, fault-tolerance, distributed-systems, dynamic-typing, web-backend
