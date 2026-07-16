# ggml-org/llama.cpp

> LLM inference in C/C++ — the reference engine for running quantized language models locally on commodity hardware.

[GitHub repo](https://github.com/ggml-org/llama.cpp) ·
[Official website](https://llama.app) ·
[License: MIT](https://github.com/ggml-org/llama.cpp/blob/master/LICENSE)

## Overview

llama.cpp is a C/C++ implementation of large-language-model inference, started by Georgi Gerganov in March 2023 as a weekend project to run Meta's LLaMA weights on a MacBook without a GPU or a Python stack[^1]. It grew into the de facto local-inference runtime: the code most consumer-facing tools (Ollama, LM Studio, Jan, KoboldCpp, GPT4All, text-generation-webui) actually shell out to or embed under the hood. The repository was later moved from `ggerganov/llama.cpp` to the `ggml-org` organization as the project professionalized around Gerganov's ggml.ai.

The defining bet is **quantization on CPU-first, dependency-free C/C++**. Instead of assuming a datacenter GPU, llama.cpp compresses model weights to 2–8 bits and hand-writes SIMD kernels (ARM NEON, AVX/AVX2/AVX512/AMX, Apple Metal, custom CUDA) so a 7B–70B model runs on a laptop, a Raspberry Pi, or a single consumer GPU. It ships the **GGUF** file format (a single-file container for weights + metadata + tokenizer) and a family of command-line tools plus an OpenAI-compatible HTTP server.

The tension the project lives with: it moves extremely fast (multiple releases per day, no semantic versioning, a large rotating cast of backend contributors) and it is a research playground for the underlying `ggml` tensor library[^2]. That means bleeding-edge model support and quantization schemes land here first, but the API surface, build flags, and quant formats change often, and long-term stability is not a design goal.

## Getting Started

Install a prebuilt binary (Homebrew, winget, nix, conda-forge) or build from source:

```sh
# macOS / Linux
brew install llama.cpp

# or build from source with Metal (macOS) / CUDA (NVIDIA)
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build                      # add -DGGML_CUDA=ON for NVIDIA
cmake --build build --config Release -j
```

Run a model — llama.cpp can pull GGUF files straight from Hugging Face with `-hf`:

```sh
# Interactive CLI
llama-cli -hf ggml-org/gemma-3-1b-it-GGUF

# OpenAI-compatible server on :8080 (chat completions + embeddings + web UI)
llama-server -hf ggml-org/gemma-3-1b-it-GGUF --port 8080
```

```sh
# Any OpenAI SDK / curl client now works against it:
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Explain GGUF in one sentence."}]}'
```

## Architecture / How It Works

llama.cpp is a thin, model-aware layer over **ggml**, a small tensor library maintained in the same GitHub org[^2]. ggml owns the compute graph, the memory allocator, the quantization formats, and the per-backend kernels; llama.cpp owns model architectures (attention layout, RoPE variants, tokenizers), the KV cache, sampling, and the user-facing tools. New backend features are frequently prototyped in llama.cpp and upstreamed into ggml.

**Backends.** ggml compiles to a selected backend at build time: CPU (with NEON / AVX / AVX512 / AMX dispatch), Apple **Metal**, **CUDA** (NVIDIA), **HIP** (AMD ROCm), **Vulkan** (vendor-neutral GPU), **SYCL** (Intel), **MUSA** (Moore Threads), and **CANN** (Huawei Ascend). A single model can be split across CPU and GPU (`-ngl` controls how many layers are offloaded to VRAM), which is how machines with too little VRAM still run large models — at the cost of PCIe transfer overhead.

**GGUF.** The current model format, introduced in August 2023 to replace the older GGML format[^3]. GGUF is a single file holding the quantized tensors, the full hyperparameter set, and the tokenizer, so a model is portable without a companion config. Converting from Hugging Face `safetensors` is done with `convert_hf_to_gguf.py`; requantizing is done with `llama-quantize`.

**Quantization.** Beyond naive round-to-nearest, llama.cpp ships **K-quants** (`Q4_K_M`, `Q5_K_M`, …) that allocate more bits to more important weights, **I-quants** (`IQ2`–`IQ4`) using importance matrices for very low bit rates, and imported vendor formats like MXFP4 (used for gpt-oss). The quant type is the single biggest quality/size/speed lever an operator chooses.

**Tools.** The build produces separate binaries: `llama-cli` (interactive/one-shot), `llama-server` (HTTP + OpenAI-compatible API + bundled web UI), `llama-bench` (throughput/latency benchmarking), `llama-quantize`, and `llama-embedding`. Multimodal (vision) support runs through the `mtmd` subsystem and is exposed in `llama-server`[^4].

The C API (`libllama`) is what bindings target. `llama-cpp-python` and dozens of language bindings wrap it, and its changelog is tracked separately from the server REST API changelog because both break independently[^5].

## Production Notes

**No semantic versioning.** Releases are continuous, tagged by build number (`bXXXX`), often several per day. There is no LTS line. Pinning an exact release tag in production is effectively mandatory; "latest" can change kernel behavior, quant support, or CLI flags between mornings.

**GGUF/quant churn.** New quantization schemes occasionally require re-downloading or re-quantizing models, and very old GGUF files can stop loading after format revisions. Treat model files as tied to a known-good binary version.

**`-ngl` and VRAM are the real tuning surface.** Throughput is dominated by how many layers fit in VRAM versus spilling to CPU. Under-offloading silently tanks tokens/sec; over-offloading OOMs the GPU mid-generation. `llama-bench` exists precisely because the right split is hardware-specific and non-obvious.

**Context length costs memory quadratically-ish via the KV cache.** Large `-c` values allocate large KV caches; KV-cache quantization (`-ctk`/`-ctv`) trades a little quality for a lot of context headroom. Flash-attention build/runtime flags matter here.

**Build flags are load-bearing.** Getting GPU acceleration means compiling with the correct backend flag (`-DGGML_CUDA=ON`, `-DGGML_METAL=ON`, `-DGGML_VULKAN=ON`, …) and matching drivers/toolkits. A "slow" install is usually a CPU-only build that silently ignored the GPU. Prebuilt binaries target specific compute capabilities and may not cover your card.

**`llama-server` is not a hardened multi-tenant gateway.** It is single-process, has a simple slot-based concurrency model, and no built-in auth beyond an optional API key. For internet-facing or high-concurrency serving, put it behind a real proxy or use a purpose-built serving stack.

**Numerics vary by backend.** The same model and prompt can produce slightly different logits on CUDA vs Metal vs CPU; determinism across hardware is not guaranteed.

## When to Use / When Not

**Use when:**
- You want to run models locally / on-device (laptop, edge, air-gapped) with no Python or cloud dependency.
- You need broad hardware coverage — Apple Silicon, AMD, Intel, NVIDIA, or CPU-only — from one codebase.
- You want the widest, earliest model and quantization support, and a single-binary OpenAI-compatible server.
- Memory footprint matters more than peak multi-GPU throughput (consumer GPUs, low-VRAM boxes).

**Avoid when:**
- You need maximum multi-GPU datacenter throughput and continuous batching for many concurrent users — GPU-native servers win there.
- You require a stable, versioned API with rare breaking changes and long-term support guarantees.
- You want a turnkey model-management UX out of the box (pull/switch/library) rather than managing GGUF files yourself.
- Your models only ship in formats you can't or won't convert to GGUF.

## Alternatives

- ollama/ollama — wraps llama.cpp/ggml with model pulling, a library, and a daemon; use it when you want the local-LLM UX rather than the raw engine.
- vllm-project/vllm — PagedAttention GPU serving with continuous batching; use it for high-throughput, multi-GPU cloud inference of unquantized/FP models.
- huggingface/text-generation-inference — production HF serving stack; use it when you're standardized on the Hugging Face ecosystem and GPU deployment.
- mlc-ai/mlc-llm — TVM-compiled inference targeting an even broader device range including mobile and WebGPU; use it when you need compiled deployment to phones/browsers.
- ml-explore/mlx — Apple's array framework for Apple Silicon; use it when you're Mac-only and want a native training+inference stack.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Initial release | 2023-03-10 | Run LLaMA inference on CPU/Apple Silicon, built on ggml[^1]. |
| 4-bit quantization | 2023-03 | Made 7B–13B models runnable on consumer RAM. |
| K-quants | 2023-06 | Mixed-precision quant types (`Q*_K`) improving quality-per-bit. |
| GGUF format | 2023-08 | Single-file format with metadata + tokenizer, replacing GGML[^3]. |
| `llama-server` | 2023 | OpenAI-compatible HTTP server with bundled web UI. |
| Multi-backend maturity | 2024 | Vulkan, SYCL, HIP/ROCm, CANN backends alongside CUDA/Metal. |
| Move to `ggml-org` | 2024 | Repository relocated from `ggerganov/` to the ggml.ai org. |
| Multimodal in server | 2025 | Vision models exposed via `llama-server` (`mtmd`)[^4]. |
| gpt-oss / MXFP4 | 2025 | Native MXFP4 support added for OpenAI's open-weight models[^6]. |

## References

[^1]: Georgi Gerganov, llama.cpp — initial commit and README, March 2023. https://github.com/ggml-org/llama.cpp
[^2]: ggml tensor library. https://github.com/ggml-org/ggml
[^3]: GGUF format specification and introduction (replaces GGML), August 2023. https://github.com/ggml-org/ggml/blob/master/docs/gguf.md
[^4]: llama.cpp multimodal documentation. https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md
[^5]: Changelog for `libllama` API and `llama-server` REST API. https://github.com/ggml-org/llama.cpp/issues/9289
[^6]: llama.cpp discussion — gpt-oss with native MXFP4 support. https://github.com/ggml-org/llama.cpp/pull/15091

## Tags

llm-inference, cpp, quantization, ggml, gguf, local-llm, on-device-ai, cuda, metal, openai-compatible-api
