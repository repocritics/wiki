# bytecodealliance/wasm-tools

> The low-level toolbox for WebAssembly binaries — parse, validate, print, mutate, and assemble modules and components, as a CLI and a set of Rust crates.

[GitHub repo](https://github.com/bytecodealliance/wasm-tools) ·
[License: Apache-2.0](https://github.com/bytecodealliance/wasm-tools/blob/main/LICENSE-APACHE)

## Overview

wasm-tools is a Bytecode Alliance project that collects the low-level plumbing for working with WebAssembly binaries into one `wasm-tools` CLI backed by a family of independently published Rust crates[^1]. It is not a compiler and not a runtime — it operates on `.wasm` and `.wat` files themselves: validating them against the spec, converting between the binary and text formats, dumping section internals, fuzzing/mutating them, and assembling the WebAssembly Component Model. If you are building a language toolchain, a runtime, a fuzzer, or a bundler that emits or consumes wasm, this is the layer you reach for instead of hand-rolling a parser.

The reason it matters beyond the CLI is the crate ecosystem. `wasmparser`, `wat`, `wast`, `wasm-encoder`, `wit-parser`, and `wit-component` are dependencies of a large slice of the Rust wasm world — Wasmtime uses `wasmparser` for validation, and cargo-component / wit-bindgen build on the WIT crates. The CLI is largely a thin front-end that wires these crates to stdin/stdout. This is the defining tension: the CLI (`wasm-tools`) and the `wat` crate hold a stable `1.x` API contract, while every other crate is versioned `0.x` and receives a major bump on essentially every release[^2]. That is deliberate — WebAssembly is a moving standard, not a fixed foundation — but it means library consumers pin versions carefully and expect churn.

wasm-tools tracks the WebAssembly proposal process directly. Every proposal at Phase 4 or later is enabled by default in validation; pre-Phase-4 proposals (custom-page-sizes, stack-switching, shared-everything-threads, wide-arithmetic, and others) are implemented but gated behind explicit `--features` flags and may change as the proposals evolve[^1].

## Getting Started

```
$ cargo install --locked wasm-tools
```

Precompiled release binaries are also published per release, and `cargo binstall wasm-tools` fetches those instead of building from source[^1].

```sh
# Validate a binary, enabling an off-by-default proposal
$ wasm-tools validate foo.wasm --features=exception-handling

# Convert between binary and text
$ wasm-tools print foo.wasm -o foo.wat
$ wasm-tools parse foo.wat  -o foo.wasm

# Pipe subcommands: strip custom sections then inspect what remains
$ wasm-tools demangle foo.wasm | wasm-tools strip | wasm-tools objdump

# Component Model: extract the WIT interface of a component
$ wasm-tools component wit component.wasm

# Lift a core module (with embedded WIT metadata) into a component
$ wasm-tools component new my-core.wasm -o my-component.wasm \
    --adapt wasi_snapshot_preview1.reactor.wasm
```

Embedding programmatically is the recommended path for anything beyond scripting — depend on the crates directly rather than shelling out to the CLI[^1]:

```rust
use wasmparser::Validator;

fn check(bytes: &[u8]) -> anyhow::Result<()> {
    Validator::new().validate_all(bytes)?;
    Ok(())
}
```

## Architecture / How It Works

The repository is a Cargo workspace where the `wasm-tools` binary is a dispatcher over subcommands, each of which is a face on one or more crates. The crates split cleanly into two worlds:

**Core WebAssembly.**
- `wasmparser` — a zero-copy, streaming, event-driven parser and the validator. It does not build an AST; it yields section/payload events and validates incrementally, which is what makes it cheap enough to sit in a runtime's load path.
- `wat` / `wast` — the text-format side. `wat` does the one-shot text-to-binary translation; `wast` exposes the full AST and the `*.wast` script format used by the spec test suite.
- `wasmprinter` — the inverse, binary to text.
- `wasm-encoder` — a builder API for emitting well-formed binary modules, the write-side counterpart to `wasmparser`'s read side.
- `wasm-smith`, `wasm-mutate`, `wasm-shrink` — the fuzzing trio: a generator that turns arbitrary bytes into a valid module, a mutator that transforms a valid module into another valid one, and a shrinker that minimizes a module while preserving a predicate.

**Component Model / WIT.**
- `wit-parser` — parses `*.wit` interface files into a resolved type graph.
- `wit-encoder` / `wit-component` — encode WIT into the binary package format and lift/lower core modules into components (`component new`, `component embed`, `component wit`).
- `wasm-metadata` — reads and writes the `name` and `producers` custom sections.

The read side (`wasmparser`) and the write side (`wasm-encoder`) are intentionally not coupled — there is no shared IR that everything round-trips through. A tool that transforms a module reads events from `wasmparser` and emits into `wasm-encoder` by hand. This keeps each crate small and streaming, but it means "parse, edit, re-emit" is more manual than a monolithic AST library would be. `wasm-mutate` and `wit-component` are where the two sides are stitched together for you.

## Production Notes

**Crate versioning is a standing upgrade tax.** Only `wasm-tools` (the CLI) and `wat` promise `1.x` API stability. `wasmparser`, `wasm-encoder`, `wit-parser`, and the rest are `0.x` and get a major bump on nearly every release, without an automatic compatibility guarantee[^2]. If your project depends on several of these transitively (common — Wasmtime, cargo-component, wit-bindgen all pull them in), a wasm-tools bump can force a coordinated update across your dependency tree, and two crates in the same build resolving to different `wasmparser` majors produces type-mismatch errors at the boundary.

**Feature flags gate correctness, not just capability.** `validate` enables all Phase-4+ proposals by default and rejects modules using pre-Phase-4 features unless you pass `--features=<name>`. The inverse also bites: a module that a newer toolchain produced with a now-default feature will fail against an older wasm-tools that predates that proposal's promotion. Pin the CLI version in CI if you validate as a gate.

**No fixed release cadence.** Releases are as-needed, triggered manually by a maintainer via a GitHub Actions workflow[^1]. There is no LTS line; patch releases against older versions require manually cutting a `release-N` branch from the tag. Do not assume a predictable version stream when planning around it.

**`wasm-compose` is deprecated.** The `wasm-tools compose` subcommand and its crate are on the way out; component composition is moving to other tooling (WAC and the component model's own composition story). Do not build new pipelines on it.

**C/C++ bindings are partial.** The `crates/c-api` CMake bindings expose only a subset of functionality[^1]. Non-Rust consumers should verify the specific calls they need exist before committing.

**Streaming parser, streaming mistakes.** Because `wasmparser` is event-driven with no AST, error handling is per-payload and it is easy to under-validate if you consume events without running the `Validator`. Use `validate_all` unless you have a specific reason to hand-drive validation.

## When to Use / When Not

**Use when:**
- You are building a wasm toolchain, runtime, bundler, or fuzzer and need spec-accurate parse/validate/encode primitives in Rust.
- You need to inspect, convert, strip, or debug `.wasm`/`.wat` files from the command line.
- You are working with the Component Model / WIT — lifting core modules to components, extracting interfaces, or generating WIT test cases.
- You want a fuzzing corpus of valid-by-construction modules (`wasm-smith`) or test-case reduction (`wasm-shrink`).

**Avoid when:**
- You want to compile a language to wasm — this is not a compiler; use a language toolchain that emits wasm and depends on these crates.
- You want to run wasm — use Wasmtime, Wasmer, wasmi, or a browser engine.
- You need a stable long-term library ABI with rare breaking changes — the `0.x` crates churn by design.
- You only need high-level component composition — that story is in flux and `wasm-compose` is deprecated.

## Alternatives

- rustwasm/walrus — mutable AST for wasm in Rust; easier "parse, edit, re-emit" than the streaming `wasmparser`+`wasm-encoder` split, at higher memory cost. Use it when you need a whole-module editable tree.
- WebAssembly/binaryen — C++ optimizer and toolkit (`wasm-opt`, `wasm-dis`); use it when you want aggressive size/speed optimization passes rather than spec-level manipulation.
- WebAssembly/wabt — the reference C toolkit (`wat2wasm`, `wasm2wat`, `wasm-objdump`); use it for language-agnostic CLI conversion without a Rust dependency.
- bytecodealliance/wasmtime — if you actually need to execute modules, not manipulate them; it consumes `wasmparser` internally.
- bytecodealliance/wac — WebAssembly composition tooling; use it instead of the deprecated `wasm-compose` for wiring components together.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-05-19 | Repository created under the Bytecode Alliance[^3]. |
| 1.0 | 2022 | CLI and `wat` crate reach the stable `1.x` line; other crates remain `0.x`[^2]. |
| ongoing | 2022–2026 | Component Model / WIT crates (`wit-parser`, `wit-component`) added and iterated as the proposal matured. |
| latest | 2026-07-16 | Active development on `main`; ~1.8k stars, releases cut as-needed[^3]. |

## References

[^1]: wasm-tools README — installation, subcommand table, proposal support, versioning, release process, and C API notes. https://github.com/bytecodealliance/wasm-tools/blob/main/README.md
[^2]: wasm-tools README, "Versioning and Releases" — `wasm-tools`/`wat` are `1.X.Y` and API-stable; all other crates are `0.X.Y` with major bumps per release and no automatic compatibility guarantee. https://github.com/bytecodealliance/wasm-tools#versioning-and-releases
[^3]: GitHub repository metadata (bytecodealliance/wasm-tools), retrieved 2026-07-18: created 2020-05-19, last push 2026-07-16, ~1768 stars, 342 forks, Apache-2.0 (repo is triple-licensed Apache-2.0 / Apache-2.0-with-LLVM-exception / MIT). https://github.com/bytecodealliance/wasm-tools

## Tags

rust, webassembly, wasm, cli, wasm-tools, component-model, wit, parser, validator, bytecode-alliance, fuzzing, developer-tools
