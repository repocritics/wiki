# gleam-lang/gleam

> A statically typed functional language that compiles to Erlang (BEAM) and JavaScript, trading a small feature set for soundness and predictability.

[GitHub repo](https://github.com/gleam-lang/gleam) ·
[Official website](https://gleam.run) ·
[License: Apache-2.0](https://github.com/gleam-lang/gleam/blob/main/LICENCE)

## Overview

Gleam is a statically typed functional language created by Louis Pilfold, first released in 2019 and reaching a stable 1.0 in March 2024[^1]. The compiler itself is written in Rust (hence GitHub's language classification), but Gleam does not target native code: it emits Erlang source for the BEAM virtual machine and, separately, JavaScript. The language borrows syntax from Rust and Elm, and a Hindley-Milner type system with full inference from the ML family.

The defining choice is deliberate smallness. Gleam has algebraic data types, pattern matching, parametric generics, labelled arguments, and a pipe operator — and almost nothing else. There are no type classes, no higher-kinded types, no macros or metaprogramming, no null, and no exceptions in idiomatic code. This keeps the language learnable in an afternoon and the error messages (modelled on Elm's) genuinely readable, at the cost of abstraction power: patterns that Haskell or Scala express with a type class are written out explicitly in Gleam.

The audience is twofold. For the BEAM ecosystem (Erlang, Elixir) it offers static types and sound inference over the same actor-model runtime — cheap processes, supervision trees, fault tolerance. For frontend and edge work, the JavaScript backend lets the same language and type checker cover the browser, most visibly through the Lustre framework's Elm-style architecture. The central tension runs along this seam: pure Gleam runs on both targets, but any code that reaches into the host via FFI is target-specific and will not port.

## Getting Started

```bash
brew install gleam        # macOS; see gleam.run for Linux/Windows binaries
# Erlang/OTP is required for the BEAM target; Node for the JS target.
gleam new my_app
cd my_app
gleam run
```

```gleam
// src/my_app.gleam
import gleam/io
import gleam/list
import gleam/int

pub fn main() {
  [1, 2, 3, 4]
  |> list.map(fn(n) { n * 2 })
  |> list.filter(fn(n) { n > 4 })
  |> list.map(int.to_string)
  |> list.each(io.println)
}
```

Algebraic data types and exhaustive pattern matching, with Gleam's separate float operators (`*.`, `+.`):

```gleam
pub type Shape {
  Circle(radius: Float)
  Rectangle(width: Float, height: Float)
}

pub fn area(shape: Shape) -> Float {
  case shape {
    Circle(r) -> 3.14159 *. r *. r
    Rectangle(w, h) -> w *. h
  }
}
```

## Architecture / How It Works

The `gleam` binary is the whole toolchain: dependency resolution, compilation, `run`, `test`, `format`, `docs`, and a built-in language server. There is no separate build-tool ecosystem to assemble.

Compilation is a straight pipeline: parse to AST, then type-check with Hindley-Milner inference (annotations are optional and mostly used for documentation), then generate source for one of two backends:

1. **Erlang backend** — emits `.erl` files, which Gleam compiles to BEAM bytecode via the Erlang compiler. Concurrency, distribution, and fault tolerance come from OTP, surfaced through the `gleam_otp` and `gleam_erlang` packages (actors, supervisors, message passing).
2. **JavaScript backend** — emits ES modules. No BEAM, so `gleam_otp` is unavailable; concurrency follows the JS host's model.

The type system is sound and total within Gleam's own boundaries: no null, no implicit conversions, `case` expressions must be exhaustive, and errors are values (`Result(a, b)` and `Option(a)`) rather than thrown exceptions. Interop crosses the boundary through `@external` functions that name a target-specific implementation — an Erlang module/function or a JS module/export. External values enter the type system unchecked, so an incorrect external annotation is the main way to introduce an unsound type at runtime.

The `use` expression is Gleam's answer to callback nesting (the "pyramid of doom"): it desugars a trailing-callback call into flat, sequential-looking code, which is how idiomatic Gleam threads `Result`, resource acquisition, and middleware without a monad abstraction the language does not have.

Packages are published to **Hex**, the registry shared with Erlang and Elixir, so Gleam projects can depend directly on Elixir and Erlang libraries (subject to the target constraint below).

## Production Notes

**The two targets are not interchangeable.** A library declares which targets it supports. Pure Gleam works on both; anything using `@external` or `gleam_otp` works only on the target it was written for. Auditing a dependency's target support before adopting it is a real, recurring step — "it's on Hex" does not mean "it runs on JavaScript."

**Ecosystem is young.** Gleam's own standard library is intentionally small, and community packages are far thinner than Elixir's or Erlang's. In practice, gaps are filled by calling mature Elixir/Erlang libraries through FFI on the BEAM target — which is idiomatic and expected, but pulls you back into dynamically typed code at the boundary and pins you to the BEAM.

**No type classes changes how you factor code.** Behaviour that other languages get from `Ord`/`Eq`/`Functor`-style abstraction is passed explicitly (functions taking comparison callbacks, per-type helper functions). This is more verbose but keeps dispatch obvious; teams coming from Haskell/Scala should expect to write more and abstract less.

**BEAM performance profile.** On the Erlang target you inherit the BEAM's strengths (massive lightweight concurrency, soft-real-time latency, supervision-based fault recovery) and its weakness (raw single-threaded numeric throughput is well behind Rust/Go/C). Choose the target to fit the workload: BEAM for concurrent services, JS for frontends and short-lived edge functions.

**Stability is a genuine selling point.** The 1.0 release committed to no breaking changes to the language[^1], so upgrade churn is low compared to fast-moving frameworks — a notable contrast with much of the JavaScript world. The toolchain still evolves quickly, but Gleam code written against 1.0 keeps compiling.

**Hiring and tooling maturity** are the honest risks: a small talent pool, fewer Stack Overflow answers, and a debugging story that often means reading generated Erlang or JavaScript when FFI or host behaviour is involved.

## When to Use / When Not

**Use when:**
- You want static types and sound inference on the BEAM without leaving the actor/OTP model.
- You're building fullstack with shared types — Gleam on the server (BEAM) and in the browser (JS, e.g. Lustre).
- Onboarding speed matters: the language is small enough to teach quickly, and error messages are unusually clear.
- You value long-term source stability over a large library ecosystem.

**Avoid when:**
- You need a deep, mature package ecosystem today — Elixir, Go, or Rust will have it and Gleam may not.
- Your problem is CPU-bound heavy computation — the BEAM is the wrong runtime; reach for Rust/C/Zig.
- You rely on higher-kinded types, type classes, or metaprogramming — Gleam deliberately omits all three.
- Team hiring at scale is a constraint; the Gleam labour market is small.

## Alternatives

- elixir-lang/elixir — dynamically typed, mature, huge ecosystem on the same BEAM; use it when productivity and library breadth outweigh static types.
- erlang/otp — the underlying platform itself; use it when you want raw BEAM/OTP without adding a language layer.
- elm/compiler — pure functional frontend language with the error messages Gleam emulates; use it for browser-only apps wanting a batteries-included framework.
- purescript/purescript — compiles to JavaScript with type classes and higher-kinded types; use it when you want a more powerful type system than Gleam offers on the JS target.
- roc-lang/roc — another "friendly" functional language chasing performance and simplicity; use it when you want native/fast output rather than a BEAM/JS host.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2019 | First public release; Erlang target only[^2]. |
| 0.16 | 2021 | JavaScript compile backend added (approx.). |
| 0.25 | 2022 | `use` expressions for flattening callbacks (approx.). |
| 1.0 | 2024-03 | Stable release; language stability guarantee[^1]. |

## References

[^1]: Louis Pilfold, "Gleam version 1.0.0!" — 2024-03-04. https://gleam.run/news/gleam-version-1/
[^2]: Gleam website and language tour. https://gleam.run — tour at https://tour.gleam.run

## Tags

gleam, erlang, beam, elixir, functional-programming, statically-typed, compiler, javascript, type-inference, actor-model, programming-language
