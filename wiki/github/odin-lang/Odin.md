# odin-lang/Odin

> A data-oriented C alternative: manual memory management with modern
> ergonomics, no package manager, and an implicit context system.

[GitHub repo](https://github.com/odin-lang/Odin) ·
[Official website](https://odin-lang.org) ·
[License: Zlib](https://github.com/odin-lang/Odin/blob/master/LICENSE)

## Overview

Odin is a general-purpose systems language created by Bill "gingerBill" Hall,
with development starting in 2016 (the GitHub repo went public 2016-11-23)[^1].
Its self-description — "the C alternative for the joy of programming" — is
accurate about scope: Odin does not try to solve memory safety like Rust or
compile-time metaprogramming like Zig. It keeps C's model (manual memory,
pointers, no hidden control flow) and fixes C's paper cuts: real slices,
tagged unions, `defer`, multiple return values, distinct types, a sane package
system, and zero-initialization of everything by default (the "ZII" idiom).

The defining design choice is **data-oriented pragmatism for interactive
software**. Built-in `#soa` struct layouts, array programming with swizzles
(`v.xyz`), a native `matrix` type, and a `vendor` library shipping bindings
for raylib, SDL, OpenGL, Vulkan, stb, and miniaudio make it clear who this is
for: game, tool, and simulation programmers. The flagship production proof is
JangaFX, whose commercial VFX suite (EmberGen and siblings) is written in
Odin, and where Hall works[^2].

The tradeoff to understand upfront: Odin is deliberately conservative. There
is no borrow checker, no async story, no macros, and — by explicit design —
no package manager[^3]. The compiler is self-declared "still in development"
and there is no 1.0; releases are monthly `dev-YYYY-MM` snapshots[^4].

## Getting Started

Prebuilt binaries ship with each monthly release; on macOS `brew install
odin` works, elsewhere download from GitHub releases or build from source
(requires LLVM/Clang, and MSVC on Windows)[^5].

```odin
// main.odin
package main

import "core:fmt"

Entity :: struct {
	id:   int,
	name: string,
}

main :: proc() {
	entities := []Entity{
		{id = 1, name = "player"},
		{id = 2, name = "enemy"},
	}
	for e in entities {
		fmt.printfln("%d: %s", e.id, e.name)
	}
}
```

```bash
odin run .          # build + run the package in the current directory
odin build . -o:speed   # optimized build
```

## Architecture / How It Works

The compiler is written in C++ and emits machine code through **LLVM**. There
is no bootstrapped self-hosted compiler and no interpreter; `odin` is a
single binary handling parse, type check, and codegen.

The most distinctive runtime mechanism is the **implicit `context`**: every
procedure using the default Odin calling convention receives a hidden
`context` struct carrying `allocator`, `temp_allocator`, `logger`, and
assertion handlers[^6]. Core-library code allocates through
`context.allocator`, so the *caller* decides allocation strategy — swap in an
arena for a subsystem, or a tracking allocator in debug builds to report
every leak at exit. This is Odin's answer to memory management: not safety
guarantees, but making custom allocators the path of least resistance.
Procedures can opt out with the `"contextless"` convention, and `foreign`
blocks bind C libraries directly with no header translation step.

Error handling is multiple return values plus the `or_return` / `or_else`
operators — no exceptions, no `Result` monad. Packages are directories;
imports resolve against named collections (`core:`, `vendor:`, or your own),
which is also the whole dependency story: third-party code is vendored into
your tree by hand or by git submodule. Parametric polymorphism (`proc($T:
typeid)`) covers generics; there is intentionally no macro system, with
compile-time `when` blocks handling platform variance.

## Production Notes

- **No stability guarantee.** Monthly `dev-` releases can and do change
  language and `core:` APIs. Teams shipping on Odin pin a release and
  upgrade deliberately; read the monthly release notes before bumping.
- **No package manager is a real cost.** Fine for the intended style
  (few dependencies, vendor everything), painful if you expect an
  ecosystem of small composable libraries. Third-party library discovery
  happens largely via the Odin Discord and awesome-lists.
- **Memory bugs are your problem.** Use-after-free and out-of-bounds writes
  compile fine. Mitigations are idiomatic rather than enforced: tracking
  allocator in debug builds, arenas to make lifetimes coarse, and
  `free_all(context.temp_allocator)` once per frame — forgetting that last
  one is the classic Odin leak.
- **Compile times are LLVM-bound.** Debug builds of medium projects are
  quick; `-o:speed` builds are noticeably slower. There is no incremental
  compilation.
- **Tooling is community-grade.** The language server (DanielGavin/ols)[^7]
  plus editor plugins give completion and go-to-definition, but expect less
  polish than rust-analyzer or gopls. Debugging is a strength for the
  domain: the compiler emits standard PDB/DWARF, so RAD Debugger, Visual
  Studio, and LLDB work on Odin binaries directly.
- **Windows is first-class**, reflecting the game-tooling audience; Linux
  and macOS are well supported, with additional targets (WASM, FreeBSD,
  ARM) of varying maturity.
- Activity is healthy as of mid-2026: pushes within days, ~11.4k stars,
  ~1k forks, and 779 open issues — a real but niche community, roughly a
  third of Zig's by stars.

## When to Use / When Not

**Use when:**
- You write games, engines, tools, or simulations and want C's control with
  fewer footguns and batteries included (`core:` + `vendor:`).
- Your team already thinks in arenas and data-oriented layouts; Odin makes
  that style the default rather than a discipline.
- You need painless C interop in both directions without a binding
  generator.

**Avoid when:**
- You need compiler-enforced memory safety or untrusted-input hardening —
  that is Rust's territory, not Odin's.
- You depend on a large third-party package ecosystem or a package manager.
- You need long-term language stability commitments (pre-1.0, monthly
  breaking changes possible).
- Your domain is networked services with heavy concurrency — Odin has
  threads and sync primitives but no async runtime story comparable to
  Go or Tokio.

## Alternatives

- ziglang/zig — use instead when you want comptime metaprogramming, a
  bundled C/C++ cross-compilation toolchain, and a larger community.
- rust-lang/rust — use instead when memory safety must be enforced by the
  compiler rather than by convention.
- c3lang/c3 — use instead when you want a smaller, closer-to-C evolution
  with C ABI compatibility as the headline feature.
- golang/go — use instead when a GC is acceptable and the workload is
  services rather than interactive/native software.
- Jai (Jonathan Blow; closed beta, no public repo) — the contemporaneous
  game-focused C successor; Odin is the one you can actually download.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-11-23 | GitHub repository created; development began earlier in 2016[^1]. |
| 0.x series | 2017–2021 | Early tagged versions (through 0.9.x) while the language design settled. |
| dev-2021-05 | 2021-05 | Switch to monthly `dev-YYYY-MM` rolling releases[^4]. |
| dev-2026-07a | 2026-07-10 | Latest release at time of writing; monthly cadence ongoing[^4]. |

## References

[^1]: odin-lang/Odin repository, created 2016-11-23. https://github.com/odin-lang/Odin
[^2]: JangaFX — EmberGen and related tools are written in Odin. https://jangafx.com/
[^3]: Odin FAQ, including the position on package managers. https://odin-lang.org/docs/faq/
[^4]: Odin GitHub releases (monthly `dev-YYYY-MM` tags since 2021-05). https://github.com/odin-lang/Odin/releases
[^5]: Odin installation docs. https://odin-lang.org/docs/install/
[^6]: Odin language overview — implicit context system, allocators. https://odin-lang.org/docs/overview/
[^7]: ols — Odin language server. https://github.com/DanielGavin/ols

## Tags

odin, systems-programming, programming-language, compiler, data-oriented-design, manual-memory-management, game-development, c-alternative, llvm
