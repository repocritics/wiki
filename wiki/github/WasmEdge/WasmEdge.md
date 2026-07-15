# WasmEdge/WasmEdge

> A C++ WebAssembly runtime for cloud-native, edge, and AI workloads, with an LLVM AOT compiler and non-standard host extensions.

[GitHub repo](https://github.com/WasmEdge/WasmEdge) ·
[Official website](https://WasmEdge.org) ·
[License: Apache-2.0](https://github.com/WasmEdge/WasmEdge/blob/master/LICENSE)

## Overview

WasmEdge is a standalone WebAssembly runtime written in C++, started in 2019 as SSVM (Second State VM) by Second State and later donated to the CNCF, where it is a sandbox project[^1]. Its positioning is not "run untrusted plugins in a browser" but "run untrusted or community code as a sandboxed function inside a server, edge node, container, or blockchain node." It competes in the same slot as Wasmtime, Wasmer, and WAMR: a runtime you embed in a host process or invoke from a CLI.

The defining tradeoff is **reach versus standards-purity**. WasmEdge ships a large set of extensions beyond the WebAssembly/WASI standards — POSIX-style networking sockets, Postgres/MySQL drivers, and, most consequentially, WASI-NN for AI inference — and an LLVM-based ahead-of-time compiler that turns `.wasm` into native code for throughput close to native[^2]. This makes it capable in places other runtimes are not (notably running LLMs on-device via the LlamaEdge stack), but code written against WasmEdge's socket and NN APIs is not portable to a standards-only runtime, and the AOT path pulls in an LLVM toolchain dependency at build time.

The project's most visible use in 2025–2026 is as the execution layer under **LlamaEdge**[^3] — a separate framework that runs GGUF LLMs, Whisper speech-to-text, and Stable Diffusion through WASI-NN's llama.cpp/ggml backend, producing small, portable inference binaries. This has pulled WasmEdge's roadmap toward GenAI-on-the-edge more than generic serverless.

## Getting Started

```bash
# Install the runtime + AOT compiler + default plugins
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install_v2.sh | bash
source $HOME/.wasmedge/env
```

Run a standard Wasm module compiled from Rust/C/Swift/etc.:

```bash
# Interpreted execution
wasmedge app.wasm

# AOT: compile to native, then run the optimized artifact
wasmedgec app.wasm app_aot.wasm
wasmedge app_aot.wasm
```

Embed a Wasm function inside a Go host process (SDK):

```go
import "github.com/second-state/WasmEdge-go/wasmedge"

func main() {
    conf := wasmedge.NewConfigure(wasmedge.WASI)
    vm := wasmedge.NewVMWithConfig(conf)
    defer vm.Release()
    res, _ := vm.RunWasmFile("add.wasm", "add", uint32(2), uint32(3))
    println(res[0].(int32))
}
```

## Architecture / How It Works

WasmEdge has two execution modes over a shared loader/validator front end:

1. **Interpreter** — parses and executes bytecode directly. Fast startup, no toolchain dependency, lower steady-state throughput.
2. **AOT compiler** (`wasmedgec` / the `wasmedge/aot` component) — lowers Wasm to LLVM IR and emits a native shared object embedded back into a `.wasm` wrapper. The runtime detects the AOT section and runs native code. This is where the "fastest Wasm VM" performance claims come from[^2], but it makes builds depend on LLVM and produces platform-specific artifacts.

**Host functions and plugins** are the extension mechanism. Standard WASI is built in; everything else — WASI-NN (AI), WASI-Crypto, WASI-Logging, the socket API, image/OCR helpers, the FFmpeg and OpenCV bindings — ships as loadable plugins. WASI-NN is itself a dispatcher: the actual inference runs in a backend plugin (llama.cpp/ggml, PyTorch, TensorFlow-Lite, OpenVINO, Piper), and which backends you get depends on how the binary was built or which plugin packages you installed[^3].

**Embedding SDKs** exist for C (the canonical ABI), Rust, Go, Python, Java, and JavaScript/Node. All wrap the C API. A running host process creates a `Configure`, a `VM` or `Store`, registers host modules, then loads/validates/instantiates/executes — the standard runtime lifecycle.

**Container integration** is a first-class target: WasmEdge plugs into `containerd` via runwasi and into `crun`/`youki` as an OCI-compatible runtime, so a Wasm module can be scheduled by Kubernetes like a container. WasmEdge was one of the runtimes shipped in Docker's Wasm technical preview.

A load-bearing limitation lives in the internals: **the runtime is not thread-safe** by the maintainers' own statement[^4]. A `VM`/`Store` instance is single-threaded; concurrency is expected via multiple instances or host-side scheduling.

## Production Notes

- **AOT is not free.** `wasmedgec` needs LLVM (14–18 depending on version), and AOT artifacts are architecture-specific — you cannot ship one AOT `.wasm` to both arm64 and x86_64. Teams that want portability keep the interpreted module and AOT-compile at install time on the target host.
- **Plugin/ABI version coupling is the most common install failure.** WASI-NN and other plugins are compiled against a specific WasmEdge release and a specific backend version; a mismatch surfaces as "plugin not found" or silent backend fallback rather than a clear error. Pin the runtime and plugin versions together, and prefer `install_v2.sh` which resolves matching plugin builds.
- **GPU inference has a real dependency footprint.** LlamaEdge's "portable binary" story is true for the Wasm file, but the host still needs the CUDA/Metal-enabled ggml plugin present. The portability is at the app layer, not the whole-stack layer.
- **Not thread-safe.** Do not share a `VM` across OS threads. Pool instances or serialize calls[^4].
- **Still pre-1.0.** After six years the runtime remains on 0.x versioning; minor releases (0.13 → 0.14 → 0.15) have carried breaking C-API and plugin-ABI changes. Read release notes before bumping, especially if you maintain out-of-tree plugins.
- **`--enable-all` proposals.** Post-MVP features (SIMD, threads, tail-call, multi-memory, GC, exception-handling) are gated behind flags at differing maturity. Verify the specific proposal your toolchain emits is enabled, or modules fail validation at load.

## When to Use / When Not

**Use when:**
- You want to run LLMs or other AI models as small, sandboxed, on-device binaries — LlamaEdge on WasmEdge is the mature path here.
- You need near-native throughput from Wasm via AOT and can accept an LLVM build dependency.
- You're scheduling Wasm workloads through Kubernetes/containerd/crun and want an OCI-compatible Wasm runtime.
- You need the non-standard socket or database extensions for edge microservices.

**Avoid when:**
- You require strict WASI/component-model standards portability — Wasmtime tracks the standards more closely and is the reference for WASI.
- You need a tiny interpreter for deeply embedded MCUs — WAMR or wasm3 have far smaller footprints.
- You need shared-memory multithreading inside one runtime instance today.
- You want a battle-tested 1.0 API contract with rare breakage.

## Alternatives

- bytecodealliance/wasmtime — use instead when standards conformance, the component model, and WASI portability matter more than bundled AI/socket extensions.
- wasmerio/wasmer — use instead when you want broad language-embedding SDKs and a package registry (WAPM) with multiple compiler backends.
- bytecodealliance/wasm-micro-runtime — use instead (WAMR) for very small footprint and deeply embedded/RTOS targets.
- wasm3/wasm3 — use instead for the smallest possible interpreter on microcontrollers, where size beats throughput.
- extism/extism — use instead when you want a higher-level plugin framework across many host languages rather than a raw runtime.

## History

| Version | Date | Notes |
|---------|------|-------|
| SSVM (initial) | 2019 | Started as Second State VM, C++ Wasm runtime[^1]. |
| Renamed WasmEdge / CNCF sandbox | 2021-04 | Donated to CNCF as a sandbox project[^1]. |
| 0.9.x | 2022 | AOT maturation, expanded plugin system. |
| 0.11–0.12 | 2022–2023 | WASI-NN backends, containerd/crun integration hardened. |
| 0.13.x | 2024 | Component-model groundwork, more NN backends. |
| 0.14.x | 2024–2025 | LlamaEdge/ggml backend focus, GPU inference plugins. |
| 0.15.x | 2025 | Current 0.x line; still pre-1.0. |

## References

[^1]: CNCF project listing — WasmEdge (sandbox). https://www.cncf.io/projects/wasmedge/
[^2]: "A Lightweight Design for High-performance Serverless Computing," IEEE Software, Jan 2021. https://arxiv.org/abs/2010.07115
[^3]: LlamaEdge — GenAI inference framework on WasmEdge (WASI-NN). https://github.com/LlamaEdge/LlamaEdge
[^4]: WasmEdge README, "Integrations and management": "Currently, WasmEdge is not yet thread-safe." https://github.com/WasmEdge/WasmEdge#integrations-and-management

## Tags

webassembly, wasm-runtime, cpp, cloud-native, cncf, edge-computing, wasi, wasi-nn, llm-inference, aot-compiler, serverless, llvm
