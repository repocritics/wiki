# ngxson/wllama

> WebAssembly bindings for llama.cpp — run LLM inference entirely in the browser.

[GitHub repo](https://github.com/ngxson/wllama) ·
[Official docs](https://github.com/ngxson/wllama#readme) ·
[License: MIT](https://github.com/ngxson/wllama/blob/main/LICENSE)

## Overview

`wllama` is a WebAssembly compilation of [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) packaged as a TypeScript / JavaScript SDK[^1]. It runs quantized GGUF models entirely in the browser via WASM SIMD, with optional multi-threading via `SharedArrayBuffer` + Web Workers. The author, Xuan-Son Nguyen (ngxson), is a llama.cpp maintainer; wllama is the canonical browser packaging.

The pitch is **fully local LLM inference in the browser**: no server, no API key, no data leaves the client. As of 2025, with 1B–3B parameter models at 4-bit quantization, this produces 10–30 tokens/second on a modern laptop and ~5–15 tokens/second on a recent phone — usable for many product features (auto-complete, classification, structured extraction, chat with small models). It is **not** a replacement for cloud GPT-4-class models; quality is upper-bounded by what fits in browser memory and runs at the CPU/integrated-GPU speeds available.

Wllama's API surface mirrors llama.cpp's server endpoints (chat completion, embedding) with a JavaScript-idiomatic wrapper. It is one of the production-ready paths for browser-first AI features in 2026, alongside WebLLM (TVM-based, MLC) and Transformers.js (ONNX-based, smaller models).

## Getting Started

```bash
npm install @wllama/wllama
```

Basic usage with a 1B-parameter chat model:

```ts
import { Wllama } from "@wllama/wllama"

const CONFIG_PATHS = {
  "single-thread/wllama.wasm": "/wllama/single-thread/wllama.wasm",
  "multi-thread/wllama.wasm": "/wllama/multi-thread/wllama.wasm",
}

const wllama = new Wllama(CONFIG_PATHS)

await wllama.loadModelFromUrl(
  "https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf",
  { n_ctx: 2048 }
)

const out = await wllama.createChatCompletion(
  [{ role: "user", content: "What is 2+2?" }],
  { nPredict: 64 }
)

console.log(out)
```

Important: for multi-threading you must serve the page with COOP/COEP headers so `SharedArrayBuffer` is available:

```
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

## Architecture / How It Works

wllama compiles llama.cpp via Emscripten to WASM, with two builds:

1. **Single-thread** — works in any browser context, no special headers required. Slower but always available.
2. **Multi-thread** — uses `SharedArrayBuffer` + WASM threads + atomics. Requires the `crossOriginIsolated` global to be true (COOP/COEP headers). Significantly faster on multi-core machines.

The runtime layout:
- **WASM module** — compiled llama.cpp, exposing C-style functions.
- **JS wrapper** — TypeScript class managing module lifecycle, tokenization, sampling, and the model file cache (uses OPFS or IndexedDB).
- **Worker bridge** — for multi-threaded mode, computation runs in a Web Worker; the main thread posts requests and receives streaming tokens.

**Model loading.** GGUF files are downloaded once and cached in the browser's storage. For multi-gigabyte models, this can be a slow first-load — wllama supports streaming download with progress callbacks. Models can also be loaded from a `File` object (user-selected) or sharded across multiple URLs.

**Quantization formats.** All llama.cpp GGUF quantizations are supported (Q2_K through Q8_0, F16, F32). For browser use, Q4_K_M is the practical sweet spot — half the size of Q8 with negligible quality loss on smaller models.

**Sampling.** Standard llama.cpp sampling parameters: `temperature`, `top_k`, `top_p`, `min_p`, `repeat_penalty`, etc. Grammar-constrained sampling (GBNF) is supported, enabling reliable JSON / structured output.

**Embeddings.** wllama can produce embeddings via embedding-mode models (e.g., `nomic-embed-text`, `bge-small`). Output dimensions match the model.

## Production Notes

**The 4 GB WASM memory ceiling.** WebAssembly's 32-bit memory model caps a single WASM instance at 4 GB. This means:
- The model file + context KV cache + sampler state must fit in 4 GB total.
- Practical limits: ~3B parameter models at Q4 (~2 GB) + a few-thousand-token context.
- Memory64 (64-bit WASM) is in browser preview but not universally available in 2025.

**`crossOriginIsolated` requirement.** Multi-threading needs COOP/COEP headers. Many embedding contexts (iframes, third-party CDN scripts) cannot set these. Single-thread fallback is the only option there, and it is 3–5× slower.

**Mobile Safari quirks.**
- iOS Safari historically capped WASM memory at ~1 GB per origin, even on devices with much more RAM.
- iOS 17+ relaxed this somewhat but it remains the tightest constraint among major browsers.
- WebAssembly SIMD support is in iOS 16.4+, multi-threading in iOS 17+.

**Cold load cost.** Even with caching, first download of a 1B Q4 model is ~600 MB. Use shard splitting and resumable downloads. Browser storage quotas vary (typically 60% of disk, but origin-limited).

**Sampling latency.** First token (prompt eval) is slow for large prompts because tokenization + KV cache fill happens on every call. Subsequent tokens (generation) are fast. Design prompts to be short; reuse the wllama instance across requests to amortize.

**OPFS vs IndexedDB.** Newer browsers (Chrome 102+, Safari 17+) support Origin Private File System (OPFS) which is much faster than IndexedDB for large blobs. wllama uses OPFS when available, falls back to IndexedDB.

**Bundle size.** The wllama JS wrapper is a few hundred KB; the WASM blob is ~1 MB. The model itself dominates — plan for `prefetch` strategies.

**Comparison vs WebLLM.** WebLLM (MLC AI) uses TVM-compiled WebGPU kernels and gets dramatically better throughput on supported GPUs (50–100 tok/s for similar models). The cost: WebGPU is required (still patchy in Safari and older Chrome), models must be specifically compiled for WebLLM, and the toolchain is heavier. wllama runs on plain CPU + SIMD anywhere a modern browser exists.

## When to Use / When Not

**Use when:**
- You want truly local inference (privacy, offline, no API cost).
- You're targeting CPU baseline (don't want to require WebGPU).
- You need llama.cpp's full GGUF + sampling + grammar feature set.
- Your model fits in ~3 GB and your context window in a few thousand tokens.

**Avoid when:**
- You need GPT-4-class quality — small browser-runnable models cannot match.
- WebGPU is available and throughput matters more than compatibility (use WebLLM).
- Your model is too large for the WASM 4 GB limit (no current fix beyond Memory64).
- You need to run inference on mobile Safari with strict memory.

## Alternatives

- **WebLLM** (MLC AI) — WebGPU-accelerated, much faster on supported GPUs, narrower compatibility.
- **Transformers.js** (Xenova) — ONNX Runtime Web, broader model zoo, slower on LLMs.
- **ONNX Runtime Web** — generic runtime, more setup, more flexible.
- Server-side inference — vLLM, llama.cpp server, Ollama. Different tradeoff (no local-only privacy).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2024-01 | Initial release, single-thread only. |
| 0.5 | 2024-04 | Multi-thread support, OPFS caching. |
| 1.0 | 2024-08 | Stable API, OpenAI-compat chat endpoint. |
| 1.5 | 2025-01 | Improved sampler, grammar-constrained generation. |
| 1.x | 2025 | Continued llama.cpp upstream tracking. |

## References

[^1]: Xuan-Son Nguyen (ngxson), wllama README. https://github.com/ngxson/wllama
