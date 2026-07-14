# mlc-ai/mlc-llm

> Compile-once, run-anywhere LLM inference: a TVM-based ML compiler and deployment engine that targets browser, phone, and datacenter GPU from one stack.

[GitHub repo](https://github.com/mlc-ai/mlc-llm) ·
[Official website](https://llm.mlc.ai/) ·
[License: Apache-2.0](https://github.com/mlc-ai/mlc-llm/blob/main/LICENSE)

## Overview

MLC LLM is a machine-learning compiler and inference engine for large language models, first released in 2023 by the MLC team behind Apache TVM (Tianqi Chen et al.)[^1]. Where most inference stacks are hand-written kernels for one hardware family, MLC LLM takes model definitions expressed in TVM's Relax IR and *compiles* them — with quantization and target-specific code generation — into a portable model library that runs on the same engine across CUDA, ROCm, Vulkan, Metal, OpenCL, and WebGPU/WASM[^2].

The defining bet is compilation as the portability layer. The runtime, called MLCEngine, exposes an OpenAI-compatible API through REST, Python, JavaScript, iOS, and Android bindings — all backed by the same compiler[^3]. This is what lets the project claim genuinely universal reach: the identical Llama or Qwen build runs in a browser tab (WebGPU), on an iPhone's Metal GPU, on an Android Adreno GPU via OpenCL, and on a datacenter NVIDIA card via CUDA.

The tradeoff is the compile step and the narrower on-ramp. Running a model is not "point at a GGUF file"; a model architecture must exist in MLC's model zoo (or be ported to Relax), weights must be converted and quantized, and a device-specific library compiled. In exchange for that friction you get one engine that reaches targets — mobile and web especially — that llama.cpp and vLLM do not cover as a unit.

## Getting Started

Install the nightly wheels (choose a build matching your accelerator, e.g. CUDA 12.3):

```bash
python -m pip install --pre -U -f https://mlc.ai/wheels \
  mlc-llm-nightly-cu123 mlc-ai-nightly-cu123
```

Chat with a prebuilt model straight from Hugging Face (the `q4f16_1` suffix is MLC's 4-bit quantization):

```bash
mlc_llm chat HF://mlc-ai/Llama-3-8B-Instruct-q4f16_1-MLC
```

Serve an OpenAI-compatible endpoint, or drive the engine from Python:

```python
from mlc_llm import MLCEngine

engine = MLCEngine("HF://mlc-ai/Llama-3-8B-Instruct-q4f16_1-MLC")
for response in engine.chat.completions.create(
    messages=[{"role": "user", "content": "What is ML compilation?"}],
    model="HF://mlc-ai/Llama-3-8B-Instruct-q4f16_1-MLC",
    stream=True,
):
    for choice in response.choices:
        print(choice.delta.content, end="", flush=True)
engine.terminate()
```

## Architecture / How It Works

MLC LLM sits on top of **Apache TVM Unity** (the Relax IR)[^2]. There is no separate "MLC runtime" reimplementation of kernels — model logic lowers through TVM to the target backend. The deployment path has three explicit stages, all driven by the `mlc_llm` CLI:

1. **`convert_weight`** — reads original (usually PyTorch/safetensors) weights, applies a quantization format (`q4f16_1`, `q4f32_1`, `q0f16`, etc.), and writes MLC-format shards.
2. **`gen_config`** — produces `mlc-chat-config.json`: tokenizer wiring, conversation template, context window, sampling defaults.
3. **`compile`** — generates a target-specific model library (`.so`/`.dll`/`.dylib`/`.wasm`/`.tar`) containing the compiled compute graph for a chosen device (`cuda`, `metal`, `vulkan`, `webgpu`, `android`, `iphone`).

```bash
mlc_llm convert_weight ./models/Llama-3-8B/ --quantization q4f16_1 -o ./dist/Llama-3-8B-q4f16_1-MLC
mlc_llm gen_config ./models/Llama-3-8B/ --quantization q4f16_1 --conv-template llama-3 -o ./dist/Llama-3-8B-q4f16_1-MLC
mlc_llm compile ./dist/Llama-3-8B-q4f16_1-MLC/mlc-chat-config.json --device cuda -o ./dist/libs/llama3-cuda.so
```

**MLCEngine** loads the compiled library plus converted weights and runs serving-grade features: continuous batching, paged KV-cache attention, speculative decoding, and grammar-constrained (JSON) output. The same engine core is what the browser (WebLLM), iOS, and Android apps embed — the surface differs, the scheduler and compiled kernels do not.

Adding a **new model architecture** means expressing it as a TVM Relax `nn.Module` in the compiler's model library, not just dropping in a weight file. This is the structural reason MLC's supported-model list trails the GGUF ecosystem: coverage is gated on someone writing (and maintaining) the Relax definition.

## Production Notes

**The model library is device-and-shape specific.** A library compiled for one GPU family (or a materially different context length / tensor-parallel degree) is not portable to another. In practice you either pull prebuilt libraries from the `mlc-ai` Hugging Face org or run a CI step that compiles per target. Treat the compile as part of your build pipeline, not a one-time dev action.

**Nightly-first release model.** There are no traditional tagged, versioned releases for most consumers — the supported install path is dated nightly wheels (`mlc-llm-nightly-*`) paired with a matching `mlc-ai-nightly-*` (the TVM runtime). The two must be upgraded together; a stale `mlc-ai` against a newer `mlc-llm` is a common breakage. Pin exact nightly dates for reproducibility.

**Quantization is where quality lives or dies.** `q4f16_1` is the common default; it is aggressive 4-bit and can noticeably degrade some models versus `q0f16`/`q4f32_1`. There is no single "best" — validate output quality on your task before shipping the smallest format that fits the device.

**Datacenter throughput is not the design center.** For pure server-side NVIDIA serving at high concurrency, vLLM and TensorRT-LLM generally lead on tokens/sec because they hand-tune for that one target. MLC's advantage is breadth (mobile, web, non-NVIDIA GPUs), not peak datacenter numbers.

**Mobile/web footguns.** Browser (WebGPU) deployments are memory-bound by the tab and by WebGPU buffer limits; large models simply will not load. On Android, OpenCL driver quality on Adreno/Mali varies by device and is the usual source of "works on my phone, crashes on theirs." App binary size and first-load weight download are real product constraints, not afterthoughts.

**Debugging spans two projects.** Because the runtime is TVM, some failures (codegen, kernel, memory) surface as TVM errors and are resolved in the `apache/tvm` tree, not `mlc-llm`. Expect to read TVM internals for the hard cases.

## When to Use / When Not

**Use when:**
- You need one engine to reach browser, iOS, Android, and desktop/server GPUs — especially non-NVIDIA (AMD, Intel, Apple, Adreno/Mali).
- You want on-device / in-browser LLM inference with WebGPU or Metal and OpenAI-compatible ergonomics.
- Portability across heterogeneous hardware matters more than squeezing peak datacenter throughput.
- You are already in the TVM ecosystem or want compiler-level control over kernels and quantization.

**Avoid when:**
- Your only target is NVIDIA datacenter serving at maximum throughput — vLLM or TensorRT-LLM fit better.
- You want zero-friction "download a GGUF and run" with the widest model/quant coverage — llama.cpp is simpler.
- Your required model architecture is not in MLC's zoo and you cannot invest in porting it to Relax.
- You need a stable, semver-tagged dependency; the nightly cadence is not that.

## Alternatives

- ggml-org/llama.cpp — CPU/GPU GGUF runtime with the widest model and quant coverage and no compile step; use when you want the largest local-inference ecosystem and simplest setup.
- vllm-project/vllm — PagedAttention datacenter server; use for high-concurrency NVIDIA/AMD GPU serving where throughput is the goal.
- NVIDIA/TensorRT-LLM — peak NVIDIA datacenter performance via TensorRT; use when locked to NVIDIA and extracting maximum tokens/sec.
- pytorch/executorch — PyTorch's on-device/edge runtime; use when your mobile stack is already PyTorch-native.
- mlc-ai/web-llm — sibling project for pure in-browser WebGPU inference; use when the browser is the only target and you want a JS-first API.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo created | 2023-04-29 | Public launch as an ML-compilation LLM deployment demo (iOS/CLI)[^1]. |
| WebGPU / browser path | 2023 | WebLLM sibling and WASM/WebGPU targets established[^4]. |
| SLM / unified model compiler | 2024 | `mlc_llm` CLI with `convert_weight`/`gen_config`/`compile` workflow. |
| MLCEngine | 2024 | Unified OpenAI-compatible engine across server, web, iOS, Android[^3]. |
| Active development | 2026-07 | Nightly-cadence releases; repo actively pushed[^5]. |

## References

[^1]: MLC LLM project blog and documentation. https://blog.mlc.ai/ · https://llm.mlc.ai/docs/
[^2]: Apache TVM Unity / Relax — the compiler substrate MLC LLM lowers through. https://tvm.apache.org/ · Chen et al., "TVM: An Automated End-to-End Optimizing Compiler for Deep Learning," OSDI 2018. https://www.usenix.org/conference/osdi18/presentation/chen
[^3]: MLCEngine — universal LLM deployment engine, OpenAI-compatible API. https://llm.mlc.ai/docs/deploy/mlc_chat_config.html · https://blog.mlc.ai/
[^4]: WebLLM — in-browser inference sibling project. https://github.com/mlc-ai/web-llm
[^5]: GitHub repository metadata (stars, license, last push) — fetched 2026-07-15. https://github.com/mlc-ai/mlc-llm

## Tags

llm, inference-engine, ml-compiler, tvm, quantization, webgpu, cuda, metal, on-device, edge-ai, openai-compatible, python
