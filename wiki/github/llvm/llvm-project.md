# llvm/llvm-project

> A modular toolkit of compiler and toolchain libraries built around a common intermediate representation — the backend under Clang, Rust, Swift, and most modern language implementations.

[GitHub repo](https://github.com/llvm/llvm-project) ·
[Official website](https://llvm.org) ·
License: Apache-2.0 WITH LLVM-exception

## Overview

LLVM began as Chris Lattner's research project at the University of Illinois around 2000 and released LLVM 1.0 in 2003[^1]. Its founding idea is a well-specified, typed, SSA-based intermediate representation (LLVM IR) that sits between language frontends and machine backends, so that optimization and code generation can be written once and reused across many source languages and target architectures. It is the compiler infrastructure behind Clang (C/C++/Objective-C), and — out of tree — behind Rust, Swift, Julia, and a large fraction of GPU, ML, and DSL compilers.

The defining characteristic is that LLVM is a set of libraries, not a compiler. `clang`, `opt`, `llc`, and `lld` are thin drivers over reusable APIs; the intended consumers are other tool authors. This is the source of both its reach and its cost: the C++ API is deliberately not stable across releases, so anyone embedding LLVM signs up for churn on every six-month major version.

Since 2019 the whole project lives in a single "monorepo" (this repository), consolidating what were previously separate SVN repositories for LLVM core, Clang, LLD, libc++, compiler-rt, LLDB, and more[^2]. The 2019 move also relicensed the project from the University of Illinois/NCSA license to Apache-2.0 with a runtime-library exception (the "LLVM exception"), which permits linking the runtime libraries into shipped binaries without attribution[^3].

## Getting Started

Most users should install a prebuilt toolchain rather than build from source:

```bash
# Debian/Ubuntu — official apt.llvm.org packages
wget https://apt.llvm.org/llvm.sh && sudo bash llvm.sh 20

# macOS
brew install llvm

# Then compile C++ with Clang
clang++ -O2 -std=c++20 main.cpp -o main
```

Inspect the IR that drives everything else:

```bash
# Emit textual LLVM IR instead of an object file
clang -O2 -S -emit-llvm hello.c -o hello.ll

# Run an optimization pass over IR by hand
opt -passes='mem2reg,instcombine' hello.ll -S -o out.ll
```

Building from source (only if you are developing LLVM itself or need a custom config):

```bash
git clone https://github.com/llvm/llvm-project.git
cmake -S llvm-project/llvm -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_USE_LINKER=lld
ninja -C build
```

## Architecture / How It Works

LLVM's classic design is a three-phase pipeline with IR as the narrow waist:

1. **Frontend** — parses a source language and lowers it to LLVM IR. Clang is the in-tree C-family frontend; Flang handles Fortran. Rust, Swift, and Julia are out-of-tree frontends that link the LLVM libraries.
2. **Middle-end (`opt`)** — a target-independent optimizer that runs analysis and transform passes over IR (inlining, GVN, loop transforms, vectorization). Managed by the "new pass manager", which replaced the legacy pass manager as the default around LLVM 13.
3. **Backend (`llc`)** — the target-dependent code generator: instruction selection (SelectionDAG or the newer GlobalISel), register allocation, scheduling, and emission to assembly or object code.

**LLVM IR** is the contract between phases: strongly typed, in static single assignment form, with three isomorphic encodings (textual `.ll`, in-memory, and bitcode `.bc`). Bitcode has an auto-upgrade path so older files can be read by newer releases, but the IR is explicitly not a stable interchange format across major versions.

**TableGen** is a domain-specific language used pervasively inside the backends to describe registers, instructions, calling conventions, and patterns; large parts of each target are generated from `.td` files.

Notable components in the monorepo beyond core LLVM and Clang: **LLD** (a fast, mostly drop-in linker), **libc++/libc++abi** (the C++ standard library), **compiler-rt** (builtins and the sanitizer runtimes — ASan, TSan, UBSan, MSan), **LLDB** (debugger), **MLIR** (a multi-level IR framework for building domain-specific compilers, now central to ML/accelerator work), **BOLT** (post-link binary optimizer), **Polly** (polyhedral loop optimization), and **OpenMP** and **offload** runtimes.

## Production Notes

**Building it is a heavy job.** A full `Release` build of LLVM + Clang can take tens of minutes to hours and consume large amounts of RAM, and `Debug` builds are far worse — linking debug binaries frequently OOMs machines with 16 GB. The standard mitigations: link with `lld` (`-DLLVM_USE_LINKER=lld`), use `-DLLVM_TARGETS_TO_BUILD=X86` (or `Native`) instead of all targets, enable split DWARF (`-DLLVM_USE_SPLIT_DWARF=ON`), build with `-DLLVM_OPTIMIZED_TABLEGEN=ON`, and prefer `-DBUILD_SHARED_LIBS=ON` for dev iteration. ThinLTO helps release builds but costs memory.

**The C++ API is unstable by policy.** Out-of-tree passes, frontends, and backends routinely break on each six-month release. Downstream projects (Rust, Swift, Mesa's shader compilers) pin to a specific LLVM version and port deliberately. If you need a stable surface, use the C API (`llvm-c/`) or the command-line tools, which change far more slowly.

**Opaque pointers** (LLVM 14–15) removed pointee types from pointer values (`i8*` became `ptr`). This was the most disruptive IR change in years and forced essentially every IR-producing frontend to migrate; code written against typed pointers needed rewriting[^4].

**Sanitizers ship here, not just the compiler.** AddressSanitizer, ThreadSanitizer, UBSan, and MemorySanitizer live in compiler-rt and are among the most widely used parts of the project in production CI, independent of whether you use Clang as your primary compiler.

**Version skew across the toolchain matters.** Mixing a Clang from one release with an LLD, libc++, or compiler-rt from another is unsupported and produces subtle failures; treat a release as a matched set. Distribution packages (`apt.llvm.org`, Homebrew) exist precisely to avoid hand-building this.

## When to Use / When Not

**Use when:**
- You are building a compiler, JIT, or language and want a production-grade optimizer and multi-target backend rather than writing codegen yourself.
- You need best-in-class sanitizers, or a fast linker (LLD), or a well-tested libc++.
- You want cross-compilation to many architectures from one toolchain.
- You are building a domain-specific or ML compiler and can adopt MLIR.

**Avoid when:**
- You want fast compile times above all — LLVM's optimizer is heavy; TCC or Cranelift compile far faster with fewer optimizations.
- You need a stable embedding API that won't move for years — the C++ API breaks every release.
- You only need to compile C/C++ and are happy with GCC's output and licensing.
- Your target is tiny/exotic and unsupported — writing a new backend is a large, ongoing commitment.

## Alternatives

- gcc-mirror/gcc — the mature GPL alternative; often competitive or better on some optimizations and targets, but monolithic and much harder to embed as a reusable library.
- bytecodealliance/wasmtime — its Cranelift backend is the go-to when you want a small, fast-compiling code generator (JITs, Wasm) and can trade away peak optimization.
- TinyCC — use when you want an extremely small, near-instant C compiler and do not care about optimization quality.
- emscripten-core/emscripten — not an alternative but a consumer; use it when the specific goal is compiling C/C++ to WebAssembly/JS (it is built on LLVM).

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2003-10 | First public release out of UIUC[^1]. |
| 2.6 | 2009-10 | Clang included as a supported frontend. |
| 3.0 | 2011-12 | New IR type system; Clang matures for C++. |
| (relicense) | 2019 | Move to Apache-2.0 WITH LLVM-exception; monorepo consolidation[^2][^3]. |
| 14.0 | 2022-03 | Opaque pointers introduced as opt-in[^4]. |
| 15.0 | 2022-09 | Opaque pointers become the default. |
| 18.0 | 2024-03 | Six-month release; continued GlobalISel and MLIR progress. |
| 19.0 | 2024-09 | Six-month release. |
| 20.0 | 2025-03 | Six-month release. |

## References

[^1]: Chris Lattner, "LLVM" (project history) — https://llvm.org/pubs/2004-01-30-CGO-LLVM.html
[^2]: LLVM monorepo documentation — https://llvm.org/docs/GettingStarted.html
[^3]: LLVM Developer Policy, "Relicensing" and the LLVM exception — https://llvm.org/docs/DeveloperPolicy.html#new-llvm-project-license-framework
[^4]: LLVM opaque pointers documentation — https://llvm.org/docs/OpaquePointers.html

## Tags

c-plus-plus, compiler, toolchain, llvm-ir, clang, code-generation, optimizer, linker, mlir, systems-programming
