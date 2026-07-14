# nim-lang/Nim

> A statically typed, compiled systems language with Python-like syntax, compile-time macros, and no mandatory garbage collector.

[GitHub repo](https://github.com/nim-lang/Nim) ·
[Official website](https://nim-lang.org) ·
[License: MIT](https://github.com/nim-lang/Nim/blob/devel/copying.txt)

## Overview

Nim is a compiled systems programming language created by Andreas Rumpf, first
released publicly in 2008 under the name **Nimrod** and renamed to Nim in 2014[^1].
It combines an indentation-based, Python-like surface syntax with an ahead-of-time
compilation model that emits C (and optionally C++, Objective-C, or JavaScript)
which is then handed to a native C compiler. The result is a language that reads
like a scripting language but produces small, dependency-free native binaries with
performance in the same class as C.

The defining feature is **compile-time metaprogramming**. Nim exposes its own AST
to user code through `template` and `macro` constructs, so large amounts of what
would be language built-ins elsewhere (async/await, the `=>` closure sugar, most of
`std/json` and `std/times` formatting) are implemented in the standard library
rather than the compiler. This makes the language unusually malleable, but it also
means error messages and tooling frequently surface generated code rather than what
the programmer wrote — the central tradeoff of the language.

Nim's defining tension is reach versus resources. It is a genuinely capable,
mature language (1.0 shipped in 2019, 2.0 in 2023) maintained by a small core team
and a modest contributor base relative to its ambition. It competes conceptually
with Rust, Go, Zig, and Crystal but has a fraction of their corporate backing, so
ecosystem depth (libraries, IDE support, hiring pool) is the practical limiter, not
the language design.

## Getting Started

```bash
# Recommended: the choosenim toolchain manager (installs nim, nimble, tooling)
curl https://nim-lang.org/choosenim/init.sh -sSf | sh
# Or via a system package manager, e.g. `brew install nim`, `apt install nim`
```

```nim
# hello.nim — compile with: nim c -r hello.nim
import std/[strformat, sequtils, sugar]

type User = object
  id: int
  name: string

let users = @[
  User(id: 1, name: "Tom"),
  User(id: 2, name: "Brad"),
]

# UFCS: `users.mapIt(...)` == `mapIt(users, ...)`
for line in users.mapIt(fmt"Hello, {it.name}"):
  echo line
```

`nim c -r file.nim` compiles and runs; `nim c -d:release file.nim` builds an
optimized binary; `nim js file.nim` targets JavaScript. `nimble` manages packages
and project builds.

## Architecture / How It Works

Nim is a **transpile-to-C** compiler at heart, not an LLVM front end:

1. **Parse / semantic pass** — source is parsed to an AST; the semantic pass runs
   generics instantiation, overload resolution, and macro/template expansion. This
   is where most of the language's power (and slow error messages) lives.
2. **Code generation** — the typed AST is lowered to C (or C++/ObjC/JS) in the
   compiler's `cgen` backend.
3. **Native compilation** — the generated C is compiled by an external C toolchain
   (gcc, clang, MSVC). Nim caches generated C in a `nimcache/` directory.

Because the compiler is written in Nim, it is bootstrapped from checked-in C
sources in the separate `csources_v3` repository[^2]; `koch` is the meta build tool
that orchestrates bootstrapping, the test suite, docs, and releases.

**Memory management is pluggable and has changed substantially over time.** Early
Nim defaulted to a deferred reference-counting garbage collector. Modern Nim
centers on **ARC** (plain reference counting with move semantics and destructors,
no cycle collector) and **ORC** (ARC plus a cycle collector), with ORC the default
since Nim 2.0[^3]. The `--mm:` switch also still offers the older `refc`, `markAndSweep`,
`boehm`, and `none` collectors. This lets the same language span GC'd application
code and GC-free embedded/real-time targets, but it means "Nim's memory model" is a
per-build decision, and some libraries assume a specific mode.

Other load-bearing internals: **`nimsuggest`** provides IDE features (completion,
goto-def) by driving the compiler in a query mode — it lives in-repo. The **effect
system** tracks exceptions and side effects (`{.raises.}`, `func` for no-side-effect
procs). Async is a library, not a keyword: `std/asyncdispatch` and the `async`/`await`
macros rewrite procedures into state machines at compile time.

## Production Notes

**The C compiler is part of your toolchain.** Nim does not ship a backend; it needs
a working gcc/clang/MSVC on the target. This simplifies cross-compilation (emit C,
compile with a cross toolchain) but means build reproducibility and CI setup inherit
all of C toolchain management. Windows without a bundled MinGW is a common first
stumble.

**Debugging crosses an abstraction boundary.** Stack traces from a release binary,
gdb sessions, and sanitizer output reference generated C symbols and line numbers in
`nimcache`, not the original Nim. `--debugger:native` and `--lineDir:on` help, but
teams should expect to occasionally read the emitted C.

**Macro-heavy code is a double-edged sword.** Compile errors inside a macro
expansion or a deeply generic call can produce long, hard-to-parse messages that
point at library internals. This is the most-cited day-to-day friction and gets
worse the more metaprogramming a codebase leans on.

**Ecosystem depth is the real constraint.** The standard library is broad, but for
many domains you will find one community package rather than several battle-tested
options, and packages vary in maintenance. Evaluate `nimble` dependencies for bus
factor before committing. IDE tooling (via `nimsuggest`) is functional but less
polished than Rust-analyzer or gopls.

**Upgrade caution across the GC transition.** Code and libraries written for the
old `refc` default may behave differently — or expose latent lifetime bugs — under
ARC/ORC. Migrating an older Nim 1.x codebase to 2.x is usually mechanical but
warrants testing the memory-management change specifically, not just the syntax.

**`std/asyncdispatch` has known ergonomics gaps** (exception propagation, cancellation,
back-pressure) that have driven alternative async libraries such as `chronos` in
performance-sensitive projects like the Nim Ethereum clients.

## When to Use / When Not

**Use when:**
- You want C-class performance and small self-contained binaries but prefer a
  high-level, Python-like syntax.
- Compile-time metaprogramming or a DSL is central to your problem (Nim's macros
  are a genuine differentiator).
- You need one language spanning application code and GC-free embedded/real-time
  targets via the `--mm` switch.
- You're targeting both native and the browser (via the JS backend) from one codebase.

**Avoid when:**
- You need a deep, redundant library ecosystem and a large hiring pool — Rust, Go,
  and the JVM/CLR win decisively on ecosystem maturity.
- Your team values crisp compiler error messages; heavy generic/macro code can
  produce noisy diagnostics.
- You need compile-time memory-safety guarantees on the level of Rust's borrow
  checker — Nim does not provide them.
- Corporate/LTS support and a large commercial backer are procurement requirements.

## Alternatives

- crystal-lang/crystal — Ruby-like syntax compiled via LLVM; use instead when you
  want static typing with a scripting feel but prefer LLVM and a Rails-adjacent culture.
- ziglang/zig — manual memory management and C interop with no hidden control flow;
  use when you want maximum control and predictability over expressiveness.
- rust-lang/rust — use when compile-time memory safety and a large ecosystem
  outweigh Nim's lighter syntax and faster ramp-up.
- golang/go — use when team scalability, tooling maturity, and simplicity matter
  more than metaprogramming or raw single-thread performance.
- vlang/v — a much younger language in the same "simple compiled language" niche;
  consider only after weighing its maturity claims skeptically.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.10.2 | 2014-12 | Language renamed from Nimrod to Nim[^1]. |
| 1.0.0 | 2019-09-23 | First stable release; language and stdlib API commitment[^4]. |
| 1.2.0 | 2020-04 | ARC memory management introduced (opt-in). |
| 1.4.0 | 2020-10 | ORC cycle collector added; `--gc:orc`. |
| 1.6.0 | 2021-10 | Iterable concepts, `std/` import path convention matured. |
| 2.0.0 | 2023-08-01 | ORC becomes default; overloadable enums; improved async[^3]. |
| 2.2.0 | 2024-10 | Stability and stdlib refinements on the 2.x line. |

## References

[^1]: Nim renaming announcement (Nimrod → Nim), 2014. https://nim-lang.org/blog.html
[^2]: Bootstrapping C sources for the Nim compiler. https://github.com/nim-lang/csources_v3
[^3]: "Nim v2.0.0 released" — Nim blog, 2023-08-01. https://nim-lang.org/blog/2023/08/01/nim-v20-released.html
[^4]: "Version 1.0 released" — Nim blog, 2019-09-23. https://nim-lang.org/blog/2019/09/23/version-100-released.html

## Tags

nim, systems-programming, compiled-language, transpiler, metaprogramming, macros, c-backend, garbage-collection, static-typing, cross-compilation
