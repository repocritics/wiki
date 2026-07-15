# wasmerio/wasmer

> A WebAssembly runtime for running Wasm outside the browser — embeddable in a dozen host languages, with its own package registry and a POSIX-flavored ABI (WASIX).

[GitHub repo](https://github.com/wasmerio/wasmer) ·
[Official website](https://wasmer.io) ·
[License: MIT](https://github.com/wasmerio/wasmer/blob/main/LICENSE)

## Overview

Wasmer is a standalone WebAssembly runtime written in Rust, aimed at running Wasm
modules server-side, on the edge, and embedded inside applications rather than in
a browser. It was one of the first general-purpose Wasm runtimes (public since
2019) and reached a stable 1.0 in January 2021[^1]. The core value proposition is
sandboxed, near-native execution of untrusted code with no file, network, or
environment access unless explicitly granted, plus embedding SDKs for Rust, C/C++,
Python, Go, PHP, Ruby, and others.

Wasmer is as much a product company as an open-source project, and that shapes the
repository. Around the runtime sit a package registry (wasmer.io, successor to the
deprecated wapm), a hosted platform (Wasmer Edge), and WASIX — Wasmer's own
extension of WASI that adds threads, sockets, and `fork`/`exec`-style POSIX
semantics[^2]. WASIX is the defining tension: it makes real-world programs
(networking servers, threaded runtimes) work today, but it is non-standard and
diverges from the WASI Preview 2 / Component Model direction that the Bytecode
Alliance and wasmtime pursue. Code you compile for WASIX will not run on wasmtime,
and vice versa.

The other axis of the project is its pluggable compiler design: the same runtime
ships three code generators (Singlepass, Cranelift, LLVM) with very different
compile-time/run-time tradeoffs, selectable per use case.

## Getting Started

```sh
# Install the CLI
curl https://get.wasmer.io -sSfL | sh

# Run a package from the registry
wasmer run cowsay "hello world"
```

Embedding the runtime in Rust:

```rust
use wasmer::{Store, Module, Instance, imports, Value};

fn main() -> anyhow::Result<()> {
    let wat = r#"(module
        (func (export "add") (param i32 i32) (result i32)
            local.get 0 local.get 1 i32.add))"#;

    let mut store = Store::default();            // default compiler (Cranelift)
    let module = Module::new(&store, wat)?;
    let instance = Instance::new(&mut store, &module, &imports! {})?;

    let add = instance.exports.get_function("add")?;
    let result = add.call(&mut store, &[Value::I32(2), Value::I32(3)])?;
    assert_eq!(result[0], Value::I32(5));
    Ok(())
}
```

## Architecture / How It Works

Wasmer separates the frontend (parsing/validation), the compiler, and the engine
that produces executable artifacts. The API is built around a `Store` that owns
Wasm state and a `Module`/`Instance` pair.

**Compilers.** Three backends target different needs[^3]:

- **Singlepass** — single-pass code generation, no optimization. Very fast to
  compile, predictable compile time, resistant to compilation-bomb DoS. The usual
  choice for untrusted input and JIT-at-request-time scenarios (blockchain, FaaS).
- **Cranelift** — the balanced default. Reasonable compile speed, decent runtime
  performance. Shared lineage with wasmtime's code generator.
- **LLVM** — slowest to compile, fastest generated code. Pulls a heavy LLVM
  build/link dependency; used when steady-state throughput dominates.

**Engines.** The engine decides how compiled code is materialized — in-memory
(JIT) or serialized to a native object/artifact for ahead-of-time use. Wasmer can
`compile` a module to a `.wasmu` artifact and `create-exe` to produce a native
executable that statically links the runtime, enabling cross-compilation to other
targets.

**WASI / WASIX.** Host capabilities are provided through the WASI implementation.
WASIX layers threads, Berkeley sockets, and process primitives on top, which is
what lets full applications (e.g. a networked server or a threaded language
runtime) run unmodified. This is powerful and non-portable in equal measure.

**Registry.** `wasmer run <name>` resolves packages from the Wasmer registry,
making the CLI behave like a package runner rather than a bare `.wasm` loader.
This is a deliberate product choice and a departure from runtimes that only load
local files.

The repository is a Cargo workspace: `lib/api` (Rust embedding API), `lib/cli`,
`lib/compiler-*` (the three backends), `lib/wasix`, and `lib/c-api` (the C/C++
surface that most other-language SDKs bind against).

## Production Notes

- **Serialized artifacts are version-locked and unsafe to trust.** Precompiled
  modules (`Module::serialize` / `deserialize`) are tied to a specific Wasmer
  version and CPU target; upgrading Wasmer invalidates your cache. `deserialize`
  assumes the artifact is trusted native code — never deserialize attacker-supplied
  artifacts. Recompile on version bumps.
- **Pick the compiler deliberately.** Defaulting to Cranelift is fine, but running
  untrusted modules that you compile per request should favor Singlepass to bound
  compile time; the LLVM backend meaningfully increases your build complexity and
  binary size because of the LLVM dependency.
- **Metering is opt-in.** Gas/fuel-style resource limiting is provided by the
  Metering middleware; it is not on by default. Without it, a hostile module can
  spin indefinitely — you also need host-side timeouts.
- **WASIX is a lock-in decision.** Choosing WASIX for threads/sockets ties you to
  Wasmer as a runtime. If cross-runtime portability matters, stay on standard WASI
  and accept its narrower capability set.
- **Instantiation cost.** Near-native *steady-state* speed does not mean cheap
  startup; compilation and instantiation dominate short-lived invocations. Cache
  compiled artifacts (with the versioning caveat above) or keep instances warm.
- **API churn across majors.** The embedding API was reworked significantly at 1.0,
  again at 3.0, and around 4.0; upgrading across a major usually means touching
  `Store`/`Engine`/`Context` call sites, not just bumping a version.

## When to Use / When Not

**Use when:**

- You need to embed a Wasm sandbox in a non-browser host across several languages.
- You need POSIX-like capabilities (threads, sockets, processes) that WASIX
  provides and standard WASI does not.
- You want to run untrusted plugins/user code and can pair Singlepass + Metering.
- You want a package registry and `wasmer run` ergonomics, or Wasmer Edge hosting.

**Avoid when:**

- You want the standards-track path: WASI Preview 2 and the Component Model are led
  by wasmtime, not Wasmer.
- You are in Go and want zero-cgo embedding — wazero is a better fit.
- You need a tiny interpreter for microcontrollers/embedded — Wasmer is heavier.
- Cross-runtime portability of your guest code is a hard requirement (WASIX breaks it).

## Alternatives

- bytecodealliance/wasmtime — use when you want the Bytecode Alliance reference
  runtime and leadership on WASI Preview 2 / the Component Model.
- tetratelabs/wazero — use when you're in Go and want a pure-Go, zero-cgo embedded
  runtime with no external toolchain.
- WasmEdge/WasmEdge — use when targeting cloud-native/edge (CNCF) with its AI and
  networking extensions.
- bytecodealliance/wasm-micro-runtime — use when you need a small-footprint runtime
  with AOT for IoT/embedded targets.
- wasm3/wasm3 — use when you need the smallest possible interpreter on constrained
  hardware and can trade raw speed for size.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2019 | First public releases; early standalone Wasm runtime. |
| 1.0 | 2021-01 | Rewrite with pluggable Singlepass/Cranelift/LLVM compilers[^1]. |
| 2.0 | 2021-06 | Engine/API simplification, performance work. |
| 3.0 | 2022-12 | Run any registry package, cross-compilation, `wasmer deploy`. |
| 4.0 | 2023 | API stabilization, WASIX as the extended ABI[^2]. |
| 5.x | 2024– | Continued 5.x line; registry and Edge integration. |

## References

[^1]: Wasmer blog, "Wasmer 1.0" — January 2021. https://wasmer.io/posts/wasmer-1.0
[^2]: WASIX specification and rationale. https://wasix.org/
[^3]: Wasmer documentation, runtime and compiler backends. https://docs.wasmer.io/

## Tags

rust, webassembly, wasm, runtime, wasi, wasix, sandbox, edge, container, embeddable, compiler
