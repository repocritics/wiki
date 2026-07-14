# ziglang/zig

> A general-purpose systems language built around compile-time execution and explicit control — a C successor that refuses hidden allocations, hidden control flow, and a preprocessor.

[GitHub repo](https://github.com/ziglang/zig) ·
[Official website](https://ziglang.org) ·
[License: MIT](https://github.com/ziglang/zig/blob/master/LICENSE)

## Overview

Zig is a systems programming language and toolchain started by Andrew Kelley in 2015, with the first tagged release (0.1.0) in early 2017[^1]. It positions itself as a C replacement rather than a C++ or Rust competitor: manual memory management, no garbage collector, no borrow checker, and a small language surface you can hold in your head. Its two defining bets are **`comptime`** (arbitrary Zig code run at compile time, which replaces generics, macros, and conditional compilation in one mechanism) and a design principle of **no hidden behavior** — no hidden allocations, no hidden control flow, no operator overloading, no exceptions, no destructors.

The central tension is maturity versus ambition. Zig is still pre-1.0: every 0.x release ships intentional breaking changes to syntax, the standard library, and the build system, and there is no stability guarantee[^2]. Teams adopt it anyway because the toolchain does things no stable language ships out of the box — most notably `zig cc`, a drop-in C/C++ cross-compiler that makes Zig one of the easiest ways to cross-compile *C* code, even in projects that contain no Zig.

Note on hosting: as of late 2025 the canonical repository migrated to Codeberg; the GitHub repository is retained as a landing page and is no longer the source of truth[^3]. Numbers below (stargazers, forks) reflect the GitHub mirror and understate current activity.

## Getting Started

Install a released or nightly build from the downloads page, or via a package manager (`brew install zig`, `pacman -S zig`, `scoop install zig`). Zig ships as a single self-contained binary with no runtime dependency.

```bash
zig version
zig init            # scaffolds build.zig, build.zig.zon, src/main.zig
zig build run
```

A minimal program showing explicit allocators, `defer`, and error unions:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();          // reports leaks on exit
    const allocator = gpa.allocator();

    const buf = try allocator.alloc(u8, 32);
    defer allocator.free(buf);       // freed no matter how we exit

    std.debug.print("Hello, {s}!\n", .{"Zig"});
}
```

`comptime` used as the generics mechanism — types are values, functions can return types:

```zig
fn List(comptime T: type) type {
    return struct {
        items: []T,
        len: usize,
    };
}

const IntList = List(i32);          // instantiated at compile time
```

## Architecture / How It Works

**`comptime` is the unifying idea.** Types are first-class comptime values, so generic containers are ordinary functions that take a `type` and return a `type`. The same mechanism does what C would need macros, `#ifdef`, and templates for. There is no separate metaprogramming language — you write Zig, and the compiler evaluates the parts marked `comptime` (or forced by context) during compilation.

**Explicit allocation.** The standard library never allocates behind your back; any function that needs heap memory takes an `Allocator` parameter. This makes allocation strategy a caller decision. Common allocators: `GeneralPurposeAllocator` (debug-friendly, leak-detecting), `ArenaAllocator` (free-everything-at-once), `FixedBufferAllocator` (stack/static backing), and `page_allocator`. The testing allocator fails tests that leak.

**Error handling** is via error sets and error unions (`!T`). `try` propagates, `catch` handles, and `errdefer` runs cleanup only on the error path. Errors are values, not exceptions; there is no stack unwinding.

**The compiler.** Zig is now self-hosted — the compiler is written in Zig and bootstraps from a small C-source seed (a WASM build of an earlier compiler) rather than a C++ codebase, a transition completed around the 0.10 release[^4]. LLVM remains the default optimizing backend, but Zig has been building its own backends (notably x86_64) to speed up debug builds, enable incremental compilation, and eventually make the LLVM dependency optional.

**The toolchain is the product.** `zig cc` / `zig c++` wrap Clang and bundle libc headers and start files for many targets, so `zig cc -target aarch64-linux-gnu` cross-compiles without assembling a sysroot. `zig build` runs a `build.zig` file written in Zig itself (there is no separate build DSL), and `build.zig.zon` declares dependencies for the package manager introduced during the 0.11 era.

## Production Notes

**No stability guarantee.** This is the dominant operational cost. Each 0.x bump can rename standard-library APIs, change the build-system interface, and alter syntax. Pin an exact compiler version per project and expect real migration work on every upgrade; "track nightly" is viable only with CI that rebuilds against master frequently.

**Async is currently unavailable.** Zig had `async`/`await` in the language, but the feature was removed/disabled during the self-hosted compiler transition and is not usable in current stable releases[^2]. Any code or tutorial relying on Zig async predates that change. If you need async I/O today, you build it explicitly (event loops, manual state machines) — do not architect around language-level async landing on a schedule.

**Standard library is a moving target.** `std` is comprehensive but unstable and unevenly documented; reading the source is often the fastest path to correct usage. APIs you learned one release ago may have moved.

**Memory safety is partial and mode-dependent.** Zig is not memory-safe the way Rust is: use-after-free and data races are possible. It does provide runtime safety checks (bounds checks, integer-overflow checks, undefined-value detection) in Debug and ReleaseSafe modes; ReleaseFast and ReleaseSmall drop them for speed/size. Choosing the build mode is a safety decision, not just a performance one.

**Cross-compilation caveats.** `zig cc` cross-compilation is excellent for many targets but not magic — targets needing proprietary SDKs, unusual libc variants, or vendor toolchains still require setup. It shines most for Linux/musl and static binaries.

**Ecosystem is young.** Package management exists but the registry/discovery story is thin compared to Cargo or npm; many dependencies are pulled directly from Git URLs pinned by hash. Expect to vendor or wrap C libraries rather than find a mature Zig-native package for everything.

## When to Use / When Not

**Use when:**
- You want manual control (allocators, layout, no GC) with a smaller, more predictable language than C++ or Rust.
- You need painless cross-compilation — even for a C/C++ codebase, `zig cc` alone can justify adoption.
- You're building performance-critical infrastructure and can pin a compiler version and absorb upgrade churn (Bun, Ghostty, and TigerBeetle are real-world examples).
- You value explicitness and want to see every allocation and control-flow edge.

**Avoid when:**
- You need a stable language with a compatibility promise today — pre-1.0 breaking changes will cost you.
- You require compile-time memory-safety guarantees (use Rust).
- You depend on a rich, mature package ecosystem or extensive third-party libraries.
- You need language-level async now, or a large hiring pool familiar with the language.

## Alternatives

- [rust-lang/rust](../rust-lang/rust.md) — memory safety enforced at compile time; choose it when preventing use-after-free/data races matters more than language simplicity.
- **C (Clang/GCC)** — use when you need a frozen, universally supported standard and existing toolchains; Zig can incrementally interop with and cross-compile it.
- **C++** — use when you need the mature ecosystem, RAII, and templates, and can accept the language's size and complexity.
- **D / Nim** — other C-adjacent systems languages with GC-optional models; choose when you want higher-level features and a more settled language than pre-1.0 Zig.
- odin-lang/Odin — another explicit, no-hidden-magic systems language aimed at gamedev/native work; a close philosophical sibling to Zig without `comptime`-as-generics.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2017-02 | First tagged release[^1]. |
| 0.4.0 | 2019-04 | Zig Software Foundation era begins; toolchain maturing. |
| 0.6.0 | 2020-04 | Async/await functionality present in the language. |
| 0.10.0 | 2022-10 | Self-hosted compiler milestone; LLVM 15; async in flux[^4]. |
| 0.11.0 | 2023-08 | Package manager (`build.zig.zon`) and build-system work. |
| 0.12.0 | 2024-04 | Continued self-hosted backend and std changes. |
| 0.13.0 | 2024-06 | Incremental toolchain and build-system evolution. |
| 0.14.0 | 2025-03 | Further backend/std breaking changes. |

Version numbers are 0.x throughout; there is no 1.0 and no announced date for one. Treat every minor bump as potentially breaking.

## References

[^1]: Andrew Kelley, "Introduction to the Zig Programming Language" — 2017-02-08. https://andrewkelley.me/post/intro-to-zig.html
[^2]: Zig language reference and release notes (breaking-change policy, async status). https://ziglang.org/documentation/master/
[^3]: "Migrating from GitHub to Codeberg" — ziglang.org news (repository `homepage` field). https://ziglang.org/news/migrating-from-github-to-codeberg/
[^4]: Zig 0.10.0 release notes (self-hosted compiler). https://ziglang.org/download/0.10.0/release-notes.html

## Tags

zig, systems-programming, compiler, programming-language, c-interop, cross-compilation, comptime, manual-memory-management, llvm, toolchain, low-level
