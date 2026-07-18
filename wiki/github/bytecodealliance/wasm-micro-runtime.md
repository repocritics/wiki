# bytecodealliance/wasm-micro-runtime

> A lightweight WebAssembly runtime in C for embedded, IoT, RTOS, and trusted
> execution environments — interpreter, JIT, and AOT modes in one
> build-time-configurable codebase.

[GitHub repo](https://github.com/bytecodealliance/wasm-micro-runtime) ·
[Official website](https://bytecodealliance.github.io/wamr.dev) ·
[License: Apache-2.0 WITH LLVM-exception](https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/LICENSE)

## Overview

WebAssembly Micro Runtime (WAMR) is a standalone Wasm runtime written in C,
initiated at Intel and developed under the Bytecode Alliance since 2019[^1]. It
occupies the small end of the runtime spectrum: where wasmtime targets
server-class hardware, WAMR runs on Cortex-M microcontrollers with tens of
kilobytes of RAM, on RTOSes (Zephyr, NuttX, ESP-IDF, RT-Thread, VxWorks), and
inside Intel SGX enclaves — as well as on Linux, Windows, macOS, and Android.
The project ships three artifacts: the embeddable VMcore library, the `iwasm`
CLI runtime, and `wamrc`, an ahead-of-time compiler producing `.aot` files.

The defining design decision is that almost everything is a compile-time
option. Execution mode (classic interpreter, fast interpreter, Fast JIT, LLVM
JIT, AOT), libc flavor (built-in subset vs. WASI), threads, SIMD, GC, sockets,
and dozens of other features are CMake flags. This is how one codebase spans a
3.7 KB libc to near-native LLVM-backed execution[^2] — but it means "WAMR" is
really a family of runtimes, and the feature × mode × platform support matrix
is the first thing an adopter must internalize.

At 6,015 stars and 825 forks it has a smaller community than wasmtime, yet it
is effectively the default choice for Wasm on constrained devices. The
Technical Steering Committee seats engineers from Intel, Sony, Amazon,
Siemens, Xiaomi, Ant Group, and Midokura[^1] — a signal of industrial
deployment rather than hobbyist traction. Activity is healthy: pushes within
days as of mid-2026, with ~590 open issues reflecting the breadth of the
platform matrix as much as neglect.

## Getting Started

Build the `iwasm` CLI on Linux (macOS/Windows analogous):

```bash
git clone https://github.com/bytecodealliance/wasm-micro-runtime.git
cd wasm-micro-runtime/product-mini/platforms/linux
cmake -S . -B build && cmake --build build
./build/iwasm your_app.wasm
```

Ahead-of-time compile for faster startup and JIT-less targets:

```bash
wamrc -o your_app.aot your_app.wasm   # add --target=... to cross-compile
iwasm your_app.aot
```

Minimal C embedding via `wasm_export.h`[^3]:

```c
wasm_runtime_init();
wasm_module_t mod = wasm_runtime_load(buf, size, err, sizeof(err));
wasm_module_inst_t inst =
    wasm_runtime_instantiate(mod, 8192 /*stack*/, 8192 /*heap*/, err, sizeof(err));
wasm_function_inst_t fn = wasm_runtime_lookup_function(inst, "add");
wasm_exec_env_t env = wasm_runtime_create_exec_env(inst, 8192);
uint32_t argv[2] = {2, 3};
wasm_runtime_call_wasm(env, fn, 2, argv);  /* result in argv[0] */
```

## Architecture / How It Works

VMcore supports five running modes[^4], selected at build time:

1. **Classic interpreter** — smallest and most portable; ~56 KB text on
   Cortex-M4F[^2]. The fallback that works everywhere.
2. **Fast interpreter** — pre-translates Wasm bytecode into an internal
   representation at load time; meaningfully faster than classic at the cost
   of extra load-time memory (~59 KB text).
3. **Fast JIT** — a self-contained lightweight code generator: quick startup,
   moderate speedup, no LLVM dependency.
4. **LLVM JIT** — near-native performance, but links LLVM into the runtime.
5. **AOT** — `wamrc` compiles Wasm to a `.aot` file offline; the runtime's
   self-implemented AOT loader (~29 KB) runs it on Linux, Windows, macOS,
   Android, SGX, and bare-metal MCUs. XIP (execute-in-place) lets AOT code run
   directly from flash without copying to RAM[^5].

A dynamic tier-up mode starts modules in Fast JIT and recompiles hot code with
LLVM JIT in the background. AOT is the only codegen path on platforms that
forbid runtime code generation (SGX, most MCUs).

For host interop, WAMR offers WASI (preview 1, plus wasi-threads and wasi-nn)
or a minimal built-in libc subset for bare-metal use. Host functions are
registered through a native-symbol API; the standard `wasm-c-api` is also
implemented, and Go, Python, and Rust bindings wrap the C API. Sandboxing uses
OS guard pages for linear-memory bounds checks on 64-bit platforms and
software bounds checks where virtual memory is unavailable. Post-MVP proposals
(SIMD, reference types, bulk memory, shared memory, tail calls, GC, exception
handling, memory64) are implemented, but coverage differs per running mode —
a feature working in the interpreter is not guaranteed in Fast JIT or AOT.

## Production Notes

**Pick the running mode deliberately.** Interpreter for footprint and
portability, AOT for performance on JIT-hostile targets, LLVM JIT only where
binary size and LLVM build times are acceptable. Building `wamrc` or LLVM JIT
means building LLVM — expect long first builds and a large toolchain.

**AOT files are coupled artifacts.** `.aot` output is target-specific (pass
`--target`/`--target-abi` per architecture) and tied to the runtime's AOT ABI;
plan to recompile all modules when upgrading the runtime, and version your
`.aot` artifacts alongside the runtime in OTA pipelines.

**Build-config mismatches surface at load time.** A module using SIMD or bulk
memory fails to load on a runtime compiled without those flags, with errors
that don't obviously point at the missing CMake option. Keep the runtime's
build configuration documented next to the toolchain that produces modules.

**Memory sizing is manual.** Per-instance Wasm stack and app heap sizes are
set at instantiation; undersized stacks on MCUs produce aborts that are
painful to diagnose. The project's memory-model and tuning docs are required
reading before shipping on constrained devices[^6].

**The sandbox is written in C.** WAMR has had security advisories over its
lifetime; treat runtime upgrades as security-relevant, and track the GitHub
advisories feed if you execute untrusted modules.

**Two threading stories.** The older built-in pthread library and the
standards-track wasi-threads coexist; new work should target wasi-threads,
but toolchain support differs.

## When to Use / When Not

**Use when:**
- You need Wasm on microcontrollers, RTOSes, or bare metal where a tens-of-KB
  runtime is the budget.
- You target JIT-forbidden environments (SGX/TEE, locked-down OSes) and can
  use AOT compilation.
- You're embedding a sandbox into a C/C++ host and want a small, dependency-
  free C API.
- You need XIP or tight control of RAM at instantiation time.

**Avoid when:**
- You're on server-class hardware and want the strongest standards
  conformance and security process — wasmtime is the better default.
- You want a batteries-included component-model/WASI preview 2 story today;
  WAMR's center of gravity remains core Wasm + WASI preview 1.
- Your team won't own a cross-compile + CMake feature-flag build matrix.
- You need a managed-language embedding first (Go, JVM) rather than a C API.

## Alternatives

- bytecodealliance/wasmtime — use instead on servers/desktops when you want
  Cranelift codegen, component model, and the most active standards work.
- WasmEdge/WasmEdge — use instead for cloud-native/edge deployments wanting a
  CNCF runtime with a plugin ecosystem.
- wasmerio/wasmer — use instead for a Rust-embedded runtime with multiple
  compiler backends and a package registry.
- wasm3/wasm3 — use instead when you need an even smaller pure interpreter
  and can forgo JIT/AOT entirely (project is minimally maintained).
- tetratelabs/wazero — use instead when embedding in Go and a zero-CGo,
  pure-Go runtime matters more than raw speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-05 | Initial open-source release, initiated at Intel[^1]. |
| 1.0.0 | 2022-09 | First stable release after three years of pre-1.0 development[^7]. |
| 1.3.0 | 2023-12 | Last 1.x minor line[^7]. |
| 2.0.0 | 2024-04 | 2.x line begins; Wasm GC support lands[^7]. |
| 2.2.0 | 2024-10 | Incremental proposal and platform coverage[^7]. |
| 2.4.0 | 2025-07 | Current major line as of mid-2026[^7]. |

## References

[^1]: WAMR README — project description, TSC membership, license.
      https://github.com/bytecodealliance/wasm-micro-runtime#readme
[^2]: WAMR README, "Key features" — footprint figures (text size via bloaty,
      Cortex-M4F). https://github.com/bytecodealliance/wasm-micro-runtime#key-features
[^3]: "Embed WAMR" guide and `wasm_export.h` API.
      https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/doc/embed_wamr.md
[^4]: WAMR blog, "Introduction to WAMR running modes".
      https://bytecodealliance.github.io/wamr.dev/blog/introduction-to-wamr-running-modes/
[^5]: XIP (execute-in-place) documentation.
      https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/doc/xip.md
[^6]: Memory usage tuning guide.
      https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/doc/memory_tune.md
[^7]: WAMR GitHub releases (dates verified via GitHub API, 2026-07).
      https://github.com/bytecodealliance/wasm-micro-runtime/releases

## Tags

c, webassembly, wasm-runtime, embedded, iot, rtos, aot-compilation, jit,
interpreter, wasi, trusted-execution, bytecode-alliance
