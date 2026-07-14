# vlang/v

> A Go-flavored, compiles-to-C systems language whose real shape is more modest — and more usable — than its original marketing promised.

[GitHub repo](https://github.com/vlang/v) ·
[Official website](https://vlang.io) ·
[License: MIT](https://github.com/vlang/v/blob/master/LICENSE)

## Overview

V is a small, statically typed language created by Alex Medvednikov and open-sourced in 2019[^1]. Its main backend emits human-readable C, which is then handed to a C compiler (bundled tcc for development, gcc/clang/MSVC for production). The language reads like Go with immutability defaults: no null, no global variables, options/results instead of exceptions, `mut` required to make a binding mutable, and `vfmt` as a canonical formatter. The compiler is self-hosting and, per its headline claim, rebuilds itself in under a second[^1].

V launched with an unusually aggressive feature list — automatic C-to-V translation, an `-autofree` memory manager promising GC-free automatic frees, "as fast as C," and a translated DOOM demo — much of which was incomplete or experimental at announcement. This drew detailed public criticism and a lasting credibility gap between the website's claims and the shipping compiler. The language that actually exists in 2026 is worth judging on its own terms: a pragmatic, GC-by-default Go alternative with fast compiles and easy C interop. `-autofree` remains experimental and is not the default; the practical memory model is a Boehm GC[^2].

As of this writing V is still pre-1.0 (0.4 series) after seven years of development. The README frames 1.0 as a future "feature freeze" à la Go, with no committed date[^1]. Development is very active (the repo is pushed to daily), but the project has historically been driven by a small core with Medvednikov at the center, and the open-issue count is kept deliberately low — read that as tight triage, not as absence of rough edges.

## Getting Started

V ships no stable binary release channel; installing from source is the documented path:

```bash
git clone --depth=1 https://github.com/vlang/v
cd v
make            # builds ./v — use makev.bat on Windows
sudo ./v symlink  # optional: put `v` on PATH
v up            # self-update later
```

```v
// sum.v — run with: v run sum.v
fn main() {
	numbers := [1, 2, 3, 4, 5]
	mut sum := 0
	for n in numbers {
		sum += n
	}
	println('sum = ${sum}')
}
```

`v run file.v` compiles and executes in one step; `v file.v` produces a binary. `v .` builds a directory as a project.

## Architecture / How It Works

V is a transpiler-first compiler with multiple backends. The default and most complete path lowers V source to C, then shells out to a C compiler[^1]:

- **C backend** — the mature target. It powers everything from CLI tools to `vinix`, an OS/kernel written in V. Because output is readable C, debugging and C interop are direct, and cross-compilation piggybacks on the C toolchain.
- **Native backend** — emits machine code directly (no C compiler), fast but far less complete; suitable for simple programs.
- **JavaScript backend** — for browser/Node targets; also partial.

For development builds V downloads a prebuilt **tcc** (Tiny C Compiler) into `thirdparty/`; tcc compiles fast and optimizes almost nothing[^1]. Production builds use `-prod` with gcc/clang/MSVC for optimization. This split is why "compiles in <1s" and "as fast as C" are both cited — they describe different backends and are not simultaneously true of one build.

Memory management is configurable rather than singular[^2]: Boehm GC by default; `-gc none` for manual management; `-prealloc` for arena allocation; `-autofree` for the compile-time free insertion that was originally billed as the flagship model but is still experimental. Most real V code runs on the GC.

The standard library (`vlib`) is bundled in-repo and includes batteries the ecosystem leans on: `veb` (web framework, formerly `vweb`), a built-in ORM, `net.http`/`net.websocket` (mbedtls bundled, OpenSSL via `-d use_openssl`), and the `sokol`/`gg` graphics modules used by the bundled Tetris/2048 examples. Because vlib evolves with the compiler, upgrading V and upgrading the standard library are the same action.

## Production Notes

- **Pre-1.0, but self-described as stable.** Syntax changes are mostly auto-migrated by `vfmt`, and core API churn is concentrated in modules like `os`[^1]. Still, there is no LTS, no semver guarantee, and `v up` tracks `master` — pin a commit if you need reproducibility.
- **You are shipping a C build.** A C compiler is a hard runtime dependency of the build. tcc is fine for iteration but you want gcc/clang with `-prod` for anything performance-sensitive. Static, dependency-free binaries are achievable via the Alpine/musl Docker path with `-cc gcc -cflags -static`[^1].
- **TLS is a real footgun.** The bundled mbedtls "works everywhere" but is slower and buggier under parallel HTTPS (scrapers, REST clients, RSS readers); the README explicitly recommends `-d use_openssl` for that workload, and warns that OpenSSL on Windows is hard to get right (suggests WSL2)[^1].
- **`-autofree` is not a memory strategy to bet on.** It has been "work in progress" for years. Treat GC as the default and design around it; reach for `-gc none`/`-prealloc` only with profiling.
- **GUI/graphics apps need system dev libraries.** Building anything using `sokol`/`gg` requires X11/GL/ALSA `-dev` packages per distro[^1]; it is not self-contained.
- **Ecosystem size.** Package count and third-party library maturity are small next to Go or Rust. For niche needs you will often be reading vlib source or writing C bindings yourself.

## When to Use / When Not

**Use when:**
- You want Go-like ergonomics and compile speed but with direct, readable C interop.
- You are writing low-level software (kernels, drivers, embedded-ish tooling) and value the C backend as an escape hatch.
- You like a single opinionated toolchain: formatter, package manager, web framework, ORM, and graphics all in-tree.

**Avoid when:**
- You need a stable, versioned language with backward-compatibility guarantees today — wait for a dated 1.0.
- You need a deep third-party ecosystem or long-term hiring pool; Go, Rust, and Zig are far larger.
- Your safety requirements are strict: the "no undefined behavior" claim is still marked work-in-progress[^1], and GC-free automatic memory management (`-autofree`) is not production-ready.

## Alternatives

- golang/go — the closest philosophical sibling (simplicity, fast compiles, GC); pick it when you want maturity, tooling, and a hiring pool over V's C interop.
- ziglang/zig — use when you want no GC, no hidden control flow, and first-class C interop with manual memory control.
- nim-lang/Nim — another compiles-to-C language with richer metaprogramming; choose it when you want macros and a more established ecosystem in the same niche.
- rust-lang/rust — use when compile-time memory safety without a GC is the non-negotiable requirement and you accept the learning curve.
- odin-lang/Odin — a data-oriented, C-adjacent language; pick it for gamedev/systems work where explicit allocators matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2019-06 | Public open-source release; C backend; heavy criticism over unfinished headline features[^1]. |
| 0.2 | 2020 | Stabilization; broader vlib, tooling maturation. |
| 0.3 | 2022 | Continued language/stdlib work; pure-V compiler progress. |
| 0.4 | 2023 | Current series; sum types, generics, and web/ORM modules matured[^3]. |
| 1.0 | — | Announced as a future feature-freeze; no committed date[^1]. |

## References

[^1]: V README and website — installation, backends, memory modes, TLS guidance, and 1.0 framing. https://github.com/vlang/v · https://vlang.io
[^2]: V memory management overview (GC default, `-autofree`, `-prealloc`, `-gc none`). https://vlang.io/#memory
[^3]: V documentation (`doc/docs.md`) and changelog. https://github.com/vlang/v/blob/master/doc/docs.md · https://github.com/vlang/v/blob/master/CHANGELOG.md

## Tags

v, vlang, systems-programming, compiler, compiles-to-c, programming-language, garbage-collected, go-alternative, transpiler, gui, self-hosting
