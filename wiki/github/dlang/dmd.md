# dlang/dmd

> The reference compiler for the D programming language — famous for compile speed, written in D itself.

[GitHub repo](https://github.com/dlang/dmd) ·
[Official website](https://dlang.org) ·
[License: BSL-1.0](https://github.com/dlang/dmd/blob/master/LICENSE.txt)

## Overview

DMD (Digital Mars D) is the reference implementation of the D programming language, written and still substantially maintained by Walter Bright, who previously built Zortech C++ (the first native C++ compiler) and the Digital Mars C/C++ toolchain. D 1.0 shipped in January 2007[^1]; the language's current line is D2, versioned as DMD 2.x with releases roughly every two months. The repo also contains **druntime**, D's low-level runtime (GC, threads, TypeInfo), which was merged into the dmd monorepo in 2022 — the standard library Phobos remains a separate repo[^2].

DMD's defining tradeoff is honest and well understood in the community: it compiles very fast and generates mediocre code. Its backend is a hand-written x86/x86_64 code generator descended from the 1980s Digital Mars C++ codebase — no LLVM, no GCC. That heritage buys iteration speed (small programs compile in tens of milliseconds) at the cost of optimization quality, where DMD-produced binaries commonly run measurably slower than the same code built with LDC or GDC. The standard community advice is: DMD for development, LDC for release builds[^3].

At ~3,300 stars the repo looks small next to Rust or Go, but that undersells activity: D predates GitHub's star economy (the language is from 2001, the GitHub repo from 2011), and the repo is pushed to daily with a steady release cadence. The ~3,800 open issues figure is inflated by the migration of D's long-lived Bugzilla backlog into GitHub Issues — much of it is decades of accumulated language-design and edge-case reports, not a signal of recent neglect.

## Getting Started

```bash
# official install script (Linux/macOS) — installs dmd + dub
curl -fsS https://dlang.org/install.sh | bash -s dmd
# or: brew install dmd / apt (d-apt) / Windows installer from dlang.org
```

```d
// hello.d
import std.stdio;

void main()
{
    auto langs = ["D", "C++", "Rust"];
    foreach (i, lang; langs)
        writefln("%s: %s", i, lang);
}
```

```bash
dmd -run hello.d      # compile + execute in one step
dmd -O -release hello.d   # optimized build (still prefer LDC for production)
```

Real projects use **dub** (D's package manager, bundled with the installer): `dub init myapp && dub run`.

## Architecture / How It Works

The repo splits into `compiler/` (frontend + backend) and `druntime/`:

- **Frontend** (`compiler/src/dmd/`) — lexer, parser, semantic analysis, CTFE interpreter, template/mixin expansion. Originally C++, it was machine-converted to D and shipped self-hosted in DMD 2.069 (2015)[^4]. Crucially, this frontend is shared: **LDC** (LLVM backend) and **GDC** (GCC backend, upstreamed into GCC since GCC 9) both consume the DMD frontend, so language semantics stay identical across all three compilers[^3]. It is also consumable as a library (`dmd-fe` via dub) for tooling.
- **Backend** (`compiler/src/dmd/backend/`) — the Digital Mars code generator: its own IR ("elems" and code blocks), instruction selection, register allocation, and object-file writers (ELF, Mach-O, MS-COFF) plus its own linker-adjacent utilities. It targets x86/x86_64; an AArch64 code generator has been in development in-tree, but historically Apple Silicon and other ARM users are pointed at LDC.
- **druntime** — GC (conservative, stop-the-world by default), thread and fiber support, exception unwinding, `core.*` modules, and the C bindings the language runtime needs.

Two frontend features are architecturally notable. **CTFE** (compile-time function evaluation) runs a large subset of D in an interpreter inside the compiler, which combined with templates and `mixin` string injection makes D's metaprogramming closer to a staged language than to C++ templates. **ImportC**[^5] (DMD 2.098, 2021) embeds an actual C11 compiler in the frontend, so `import` of a `.c` file compiles it and exposes its symbols directly — an FFI story that skips binding generators, though real-world headers with heavy preprocessor use still need a separate preprocessing step.

Memory safety is opt-in and gradual: `@safe`/`@trusted`/`@system` function attributes, with stricter escape analysis behind `-preview=dip1000`. The default remains permissive — D chose migration-friendliness over Rust-style enforcement, and several attempts to flip defaults (e.g. `@safe` by default) have been proposed and withdrawn.

## Production Notes

- **Compiler choice is the first decision.** DMD is one of three compilers for the same language. Benchmarks consistently favor LDC for runtime performance (differences of 20% to 2x are commonly reported depending on workload); DMD wins on compile time. CI setups often build with both.
- **The GC is real and default.** druntime's conservative stop-the-world GC is fine for tools and services with modest heaps, but latency-sensitive code uses `@nogc` enforcement, preallocation, or the `-betterC` mode, which strips druntime entirely (no GC, no classes with vtables via druntime, no exceptions) and produces C-like binaries.
- **Release cadence vs. stability.** A new 2.x lands roughly every two months with a point release or two after. Deprecations arrive continuously; code that compiles warning-free today can emit deprecation walls two versions later. Pinning the compiler version per project (dub and the install script both support side-by-side versions) is standard practice.
- **Platform coverage is uneven.** x86_64 Linux/Windows/macOS are first-class. macOS on Apple Silicon has depended on Rosetta or LDC while the AArch64 backend matures. If you target ARM, RISC-V, WebAssembly, or embedded, plan on LDC from day one.
- **Toolchain seams.** dmd, dub, Phobos, and druntime version together but live in different repos (except druntime); mixing a system-packaged dmd with a dub-installed one is a classic source of confusing linker errors. On Windows, the historical OPTLINK-vs-MS-COFF split is resolved in modern releases (MS-COFF default), but older blog posts still mislead.
- **Issue tracker archaeology.** The GitHub issue list includes the migrated Bugzilla corpus; triage labels and issue age need reading with that history in mind before judging responsiveness.

## When to Use / When Not

**Use when:**
- You want fast native-code iteration — DMD's edit-compile-run loop is among the quickest of any AOT-compiled language.
- You are working on D itself, D tooling, or need the frontend-as-a-library.
- You value C-family ergonomics plus serious compile-time metaprogramming (CTFE, mixins) without C++ template pain.
- You need incremental C interop — ImportC lets you consume C sources without writing bindings.

**Avoid when:**
- Runtime performance of shipped binaries is the priority — build with LDC (same language, LLVM optimization).
- You target ARM/embedded/WASM — DMD's backend does not go there today.
- You need a large hiring pool or ecosystem breadth — D's library ecosystem is far smaller than Rust's, Go's, or C++'s.
- You require enforced memory safety by default — D's safety is opt-in; Rust enforces it.

## Alternatives

- ldc-developers/ldc — the same D language on LLVM; use it instead when you need optimized release builds or non-x86 targets.
- gcc-mirror/gcc — GDC is upstream in GCC; use it when you want D inside a distro-standard GCC toolchain.
- rust-lang/rust — use it when you need compiler-enforced memory safety without a GC and a much larger ecosystem.
- ziglang/zig — use it when you want C-replacement simplicity with comptime metaprogramming and no runtime.
- nim-lang/Nim — use it when you want a similar niche (GC'd, native, strong metaprogramming) with Python-like syntax.

## History

| Version | Date | Notes |
|---------|------|-------|
| D 1.0 | 2007-01 | First stable release after ~5 years of development[^1]. |
| 2.000 | 2007-06 | D2 begins: const/immutable system, later ranges and `@safe`. |
| — | 2011-01 | Source moves to GitHub (this repo). |
| 2.069 | 2015-11 | Frontend self-hosted — C++ source converted to D[^4]. |
| — | 2017-04 | Backend relicensed under Boost; DMD fully open source for the first time[^6]. |
| 2.098 | 2021-10 | ImportC introduced[^5]. |
| 2.100 | 2022-05 | Milestone release; druntime merged into this repo the same year[^2]. |
| 2.x | 2026 | Ongoing ~bimonthly releases; AArch64 backend work in progress. |

## References

[^1]: D changelog, version 1.000 — 2007-01-02. https://dlang.org/changelog/1.000.html
[^2]: dlang/dmd README — repository layout including druntime. https://github.com/dlang/dmd#readme
[^3]: D Wiki, "Compilers" — DMD/LDC/GDC comparison. https://wiki.dlang.org/Compilers
[^4]: D changelog, 2.069.0 — first self-hosted (DDMD) release. https://dlang.org/changelog/2.069.0.html
[^5]: D language specification, "ImportC". https://dlang.org/spec/importc.html
[^6]: dlang/dmd LICENSE.txt (Boost Software License 1.0). https://github.com/dlang/dmd/blob/master/LICENSE.txt

## Tags

d, dlang, compiler, programming-language, native, systems-programming, code-generation, garbage-collection, metaprogramming, ctfe
