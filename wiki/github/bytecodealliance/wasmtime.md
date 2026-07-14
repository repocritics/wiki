# bytecodealliance/wasmtime

> A standalone WebAssembly runtime built on the Cranelift code generator, and the Bytecode Alliance's reference implementation for WASI and the Component Model.

[GitHub repo](https://github.com/bytecodealliance/wasmtime) ·
[Official website](https://wasmtime.dev/) ·
[License: Apache-2.0 WITH LLVM-exception](https://github.com/bytecodealliance/wasmtime/blob/main/LICENSE)

## Overview

Wasmtime is a WebAssembly runtime you embed in a host program (Rust, C/C++, Python, .NET, Go, Ruby) or invoke as a standalone CLI. It is not a browser engine and not a container runtime; it executes `.wasm` modules and components in an isolated sandbox with a capability-based security model, either JIT-compiling them ahead of first call or loading pre-compiled artifacts. The project is governed by the Bytecode Alliance and, alongside its sister project Cranelift, serves as the de facto reference for the emerging server-side Wasm standards — WASI and the Component Model[^1].

The defining tension is standards leadership versus a moving target. Because Wasmtime tracks WebAssembly proposals as they are being written, it ships features (WASI Preview 2, the Component Model, threads, GC) before those specs are frozen. Embedders who adopt early get first access; they also inherit churn — WASI Preview 1 to Preview 2 was effectively a re-architecture of the host API surface, and code written against `wasmtime-wasi` in 2023 does not compile unchanged against the 2025 crate.

The second tension is compiler footprint. Wasmtime's fast path is Cranelift, an optimizing code generator that is fast to run but adds a large dependency and a nontrivial attack surface. The project has responded with alternative execution tiers (Winch, Pulley) for embedders who cannot afford a JIT, but the default embedding still pulls in Cranelift.

## Getting Started

```console
# CLI (Linux/macOS)
curl https://wasmtime.dev/install.sh -sSf | bash
```

```rust
// Cargo.toml: wasmtime = "37"
use wasmtime::*;

fn main() -> Result<()> {
    let engine = Engine::default();
    // A tiny module exporting a function that adds its two args.
    let module = Module::new(&engine, r#"
        (module
          (func (export "add") (param i32 i32) (result i32)
            local.get 0
            local.get 1
            i32.add))
    "#)?;
    let mut store = Store::new(&engine, ());
    let instance = Instance::new(&mut store, &module, &[])?;
    let add = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;
    println!("{}", add.call(&mut store, (2, 40))?); // 42
    Ok(())
}
```

The three core types — `Engine` (compiler + config, shareable across threads), `Module`/`Component` (compiled code), `Store` (per-execution state and instances) — are the whole mental model. Everything else is configuration on `Config`.

## Architecture / How It Works

Wasmtime is a stack of separable pieces, and understanding the seams is the key to using it well.

- **Cranelift** is the default optimizing compiler. It lowers Wasm (and Rust's own MIR, in an unrelated use) to machine code via an SSA IR and ISLE-based instruction selection. It is a separate crate tree and can be swapped out.
- **Winch** is a baseline "fast compile, slow run" single-pass compiler for cases where startup latency matters more than steady-state throughput. It is opt-in via `Config::strategy`.
- **Pulley** is a portable bytecode interpreter, added to run Wasm on architectures Cranelift does not target or in environments that forbid generating executable pages (W^X, iOS). It trades raw speed for portability and a smaller trusted computing base.
- **Compilation modes**: modules can be JIT-compiled in-process, or compiled ahead of time with `wasmtime compile` / `Engine::precompile_module` into a `.cwasm` artifact loaded via the unsafe `Module::deserialize`. AOT artifacts are tied to a specific Wasmtime version and target — they are a build product, not a portable format.

Isolation rests on linear memory bounds checks and Wasm's structural guarantees; Wasmtime layers on Spectre mitigations, guard pages, and (optionally) the virtual-memory tricks that make bounds checks nearly free on 64-bit hosts. The security model is capability-based: a module has no ambient authority. WASI grants are explicit — a preopened directory, a specific clock, a network handle — passed in at instantiation.

The **Component Model** sits above core Wasm. A component wraps one or more core modules with a typed interface described in WIT (WASM Interface Types), enabling language-agnostic composition without a shared ABI negotiated by hand. WASI Preview 2 is defined in terms of components. This is where Wasmtime is furthest ahead of the wider ecosystem and where the API is least settled.

## Production Notes

- **The API is not semver-stable in the way the version number suggests.** Wasmtime hit 1.0 in 2022 and ships a new major version roughly every month on a train schedule[^2]. A "major" bump is a calendar event, not a promise of breaking changes — but breaking changes do land in majors, and the WASI/component crates move faster than the core `wasmtime` crate. Pin versions and read release notes; do not float `wasmtime = "*"`.
- **WASI Preview 1 vs Preview 2.** Preview 1 (the `wasi_snapshot_preview1` module-level ABI) still works and is what most `wasm32-wasip1` toolchains emit. Preview 2 is component-based and richer (async, sockets, typed interfaces). They are different worlds; picking the wrong target for your guest toolchain is the most common early failure.
- **AOT artifacts are version-locked.** A `.cwasm` compiled by Wasmtime N will refuse to load on N±1. If you precompile in CI and deploy separately, the Wasmtime version must match exactly or you will get deserialize errors at runtime. `Module::deserialize` is also `unsafe` — it trusts the bytes; never load a `.cwasm` you did not produce.
- **Denial-of-service is your problem, not the sandbox's.** The sandbox stops memory-safety escapes, not runaway guests. Bound execution with fuel (deterministic instruction metering) or epoch interruption (cheap cooperative preemption via a periodically-bumped counter), and cap memory with `StoreLimits`. Neither is on by default.
- **Instance density.** For many-short-lived-instances workloads (serverless, per-request isolation), use the pooling allocator and copy-on-write memory initialization; naive `Instance::new` per request will dominate your latency in page-fault and allocation cost.
- **Cranelift is a real dependency.** The default build compiles a substantial code generator into your binary. If binary size or attack surface matters, evaluate Pulley (interpreter, no codegen) or Winch, and consider `Config::wasm_*` toggles to disable proposals you do not use.
- **Fuzzing-driven, but CVEs happen.** OSS-Fuzz runs continuously and the project has a real security policy; it has also shipped genuine advisories (a 2023 Windows unmaintained-memory issue, a stack-overflow guard-page miscompile on some targets). Track the GitHub Security Advisories feed if you run untrusted guests.

## When to Use / When Not

**Use when:**
- You need to run untrusted or semi-trusted code with a strong, capability-based sandbox and predictable resource limits.
- You want the reference implementation of WASI and the Component Model and are willing to track their evolution.
- You are embedding in Rust, or in C/Python/.NET/Go/Ruby via a maintained binding, and want AOT + JIT options.
- Plugin systems, serverless/edge function isolation, deterministic replay (via fuel), or language-agnostic component composition are your use case.

**Avoid when:**
- You need a browser or a mature in-process scripting layer — V8/JS engines or Lua are lighter for that.
- You cannot tolerate a monthly release train or an API surface that still moves in the WASI/component layers.
- Binary size is a hard constraint and none of Winch/Pulley/feature-trimming gets you there — a smaller interpreter-only runtime may fit better.
- Your guests are pure compute with no isolation requirement, where native dynamic libraries would be simpler and faster.

## Alternatives

- WasmEdge (WasmEdge/WasmEdge) — use instead when you want a runtime tuned for cloud-native/edge with an LLVM AOT backend and first-class extensions (TensorFlow, sockets).
- wasmerio/wasmer — use instead when you want multiple pluggable backends (Cranelift, LLVM, Singlepass) and a package registry (WAPM) in one product.
- wasm3 (wasm3/wasm3) — use instead when you need a tiny fast interpreter for microcontrollers and embedded targets with no JIT.
- bytecodealliance/wasm-micro-runtime (WAMR) — use instead when you need the smallest footprint interpreter/AOT for deeply embedded and IoT deployments.
- extism/extism — use instead when you want a batteries-included plugin framework (built on Wasmtime) rather than a raw runtime to integrate yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-02 | Wasmtime announced under the newly formed Bytecode Alliance. |
| 1.0 | 2022-09-20 | First stable release; adopted the monthly release-train and semver-major-per-release policy[^2]. |
| — | 2023 | Component Model tooling and WASI Preview 2 development accelerate. |
| Winch | 2023–2024 | Baseline single-pass compiler stabilized for fast startup. |
| Pulley | 2024–2025 | Portable bytecode interpreter added for JIT-less targets. |
| 37.x | 2026-07 | Recent release train; repository last pushed 2026-07-14[^3]. |

## References

[^1]: Wasmtime README and project documentation, Bytecode Alliance. https://github.com/bytecodealliance/wasmtime and https://docs.wasmtime.dev
[^2]: Wasmtime release process and stability policy. https://docs.wasmtime.dev/stability-release.html
[^3]: GitHub API metadata for bytecodealliance/wasmtime, fetched 2026-07-14: 18,341 stars, 1,768 forks, Apache-2.0 license, last push 2026-07-14. https://api.github.com/repos/bytecodealliance/wasmtime

## Tags

rust, webassembly, wasm, runtime, wasi, component-model, cranelift, jit, aot, sandbox, embedded, bytecode-alliance
