# crystal-lang/crystal

> A statically-typed, natively-compiled language with Ruby-like syntax and whole-program type inference.

[GitHub repo](https://github.com/crystal-lang/crystal) ·
[Official website](https://crystal-lang.org) ·
[License: Apache-2.0](https://github.com/crystal-lang/crystal/blob/master/LICENSE)

## Overview

Crystal is a general-purpose programming language that pairs Ruby-inspired syntax with static type checking and ahead-of-time compilation to native code via LLVM[^1]. It was started in 2012 at Manas.tech in Buenos Aires (Ary Borenszweig, Juan Wajnerman, Brian Cardiff) with an explicit goal: keep Ruby's ergonomics for writing code and C's efficiency for running it. Compatibility with Ruby is a non-goal — Crystal reuses the look and feel, not the semantics or the gem ecosystem.

The defining technical decision is **global type inference**. You rarely annotate variable or argument types; the compiler infers them for the whole program at once, including union types (a value that may be `String | Nil`) and compile-time nil-checking. This eliminates most `NoMethodError`-on-nil bugs that plague dynamic Ruby while keeping source largely annotation-free. The tradeoff is that inference is program-global rather than local: an edit in one place can change inferred types elsewhere, error messages tend to be phrased as "no overload matches" across a union, and there is no incremental compilation — the analysis runs over the whole program each build.

Crystal reached its 1.0 release in March 2021, which established a backwards-compatibility promise within the major version[^2]. It is used for CLI tooling, web backends (the Kemal, Lucky, and Athena frameworks), and infrastructure daemons where a single static binary with C-like throughput and a high-level language are both wanted. It remains a comparatively small ecosystem: development is community-driven with sustained sponsorship from 84codes, and the project is actively maintained (commits land continuously on `master`).

## Getting Started

```bash
# macOS
brew install crystal
# Linux / other: see https://crystal-lang.org/install
```

```crystal
# hello.cr
struct User
  getter id : Int32
  getter name : String

  def initialize(@id : Int32, @name : String)
  end
end

users = [User.new(1, "Tom"), User.new(2, "Brad")]
users.each { |u| puts "Hello, #{u.name}" }
```

```bash
crystal run hello.cr            # compile + run
crystal build --release hello.cr # optimized static-ish binary
```

Dependencies are managed with **Shards** (`shard.yml`), Crystal's built-in package manager:

```yaml
# shard.yml
dependencies:
  kemal:
    github: kemalcr/kemal
```

## Architecture / How It Works

The compiler (`crystal`, itself written in Crystal) runs a whole-program pipeline:

1. **Parse** — source `.cr` files → AST.
2. **Semantic analysis / type inference** — the AST is typed globally. Method calls are resolved by overload matching against inferred argument types; generics and macros are expanded here.
3. **Macro expansion** — macros run at compile time and emit Crystal source, used heavily in the standard library to avoid boilerplate (e.g. `getter`, `record`, JSON/YAML serialization mapping).
4. **Codegen** — typed AST → LLVM IR → native object code, linked against system libraries (`libgc`, `libpcre2`, `libevent`, and others).

**Runtime model.** Crystal ships a runtime with cooperative concurrency built on **Fibers** — lightweight green threads scheduled over an event loop. `spawn` starts a fiber; `Channel(T)` provides CSP-style communication similar to Go. By default the scheduler has run fibers on a single OS thread, so concurrency (interleaving I/O-bound work) is default but parallelism (using multiple cores) has for most of the language's history required opting into a separate multi-threaded runtime, historically behind the `-Dpreview_mt` compile flag and treated as experimental[^3]. Teams that assume free multicore parallelism the way they would in Go should verify the multi-threading status against the Crystal version they target.

**Memory management.** Crystal uses the Boehm–Demers–Weiser conservative garbage collector. This keeps the language high-level (no borrow checker, no manual `free`), but the GC is conservative (it scans the stack for pointer-like values) and is not swappable, which matters for latency-sensitive or very-low-footprint workloads.

**C interop** is first-class and idiomatic: you declare bindings inline with `lib`/`fun` rather than through a separate FFI layer, which is how much of the standard library wraps system C APIs.

## Production Notes

**Compile times and no incremental builds.** Because type inference is whole-program, every build reanalyzes the program; there is no incremental compilation cache the way `cargo check` or `tsc --build` provides. Iteration on a large codebase is slower than the dynamic-language ergonomics might suggest. `crystal build` without `--release` (skips heavy LLVM optimization) is the fast development path; reserve `--release` for shipping.

**Error messages.** The flip side of global inference: a type error can surface far from its cause, and union types make messages verbose ("expected `String`, got `String | Nil`"). Adding an explicit nil-check or a type restriction on a method argument is the usual way to localize the failure.

**Multi-threading is the biggest "know your version" caveat.** Do not assume the default runtime uses all cores. Confirm the parallelism/`preview_mt` situation for your exact Crystal version before architecting around multicore fibers, and load-test it — the single-threaded default is excellent for I/O-bound services but will not scale CPU-bound work across cores by itself.

**GC pauses.** The Boehm conservative GC is fine for typical web/CLI workloads but is not tunable the way JVM/Go collectors are. Hard-real-time or tight-latency-SLO systems should benchmark pause behavior rather than assume it.

**Platform maturity is uneven.** Linux and macOS are the well-trodden targets. Windows support has been built out progressively and is newer and less battle-tested; treat it as supported-but-verify, especially for native dependencies. Static linking against musl (for minimal containers) is a common deployment pattern but requires the right toolchain setup.

**Ecosystem size.** Shards has a real registry but far fewer libraries than Ruby gems, npm, or crates.io. For anything off the beaten path you may be writing C bindings or a wrapper yourself. This is the single biggest adoption risk for teams — the language is stable, the surrounding library coverage is thinner.

## When to Use / When Not

**Use when:**
- You like Ruby's syntax but want compile-time type safety and native-binary throughput.
- You're shipping CLI tools or self-contained service daemons and want a single compiled artifact.
- You want compile-time nil-checking to kill an entire class of runtime errors.
- Your workload is I/O-bound and benefits from cheap fibers + channels.

**Avoid when:**
- You need a deep third-party library ecosystem — Ruby, Go, Node, or the JVM will have the package you need.
- You require guaranteed multicore parallelism out of the box and can't gate on runtime maturity — Go or Rust are safer today.
- You need no-GC, hard-real-time control over memory — Rust or Zig fit better.
- Fast incremental builds on a very large codebase are a hard requirement.

## Alternatives

- ruby/ruby — use when you want the same syntax ergonomics plus the mature gem ecosystem and dynamic flexibility, and can accept an interpreter instead of native compilation.
- golang/go — use when guaranteed multicore parallelism, a large ecosystem, and fast compiles matter more than expressive types and Ruby-like syntax.
- rust-lang/rust — use when you need memory safety and predictable latency with no garbage collector, and can pay the borrow-checker learning cost.
- nim-lang/Nim — use when you want a similar high-level-syntax-to-native-code story with more mature multi-target support and configurable GC.
- ziglang/zig — use when you want manual memory control and C interop with no GC and no hidden runtime.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial public release | 2014-06 | First open-source release; syntax and self-hosting compiler take shape[^1]. |
| 0.x series | 2014–2020 | Long pre-1.0 period; frequent breaking changes, stdlib and tooling built out. |
| 1.0.0 | 2021-03-22 | Backwards-compatibility promise within the 1.x line established[^2]. |
| 1.x | 2021–present | Ongoing: Windows support hardening, multi-threading runtime work, stdlib growth[^3]. |

## References

[^1]: Crystal README and project goals — Ruby-like syntax, static type checking, native compilation via LLVM, inline C bindings. https://github.com/crystal-lang/crystal
[^2]: "Crystal 1.0 – What to expect" / Crystal 1.0 release — backwards-compatibility guarantee within the major version. https://crystal-lang.org/2021/03/22/crystal-1.0-what-to-expect/
[^3]: Crystal language reference — concurrency, fibers, and multi-threading (`preview_mt`). https://crystal-lang.org/reference/guides/concurrency.html

## Tags

crystal, programming-language, compiler, llvm, static-typing, type-inference, ruby-like, native-compilation, fibers, systems-programming
