# ggml-org/ggml

> A dependency-free tensor library in C/C++ for on-device machine learning inference — the engine underneath llama.cpp and whisper.cpp.

[GitHub repo](https://github.com/ggml-org/ggml) ·
[License: MIT](https://github.com/ggml-org/ggml/blob/master/LICENSE)

## Overview

ggml is a low-level tensor library written by Georgi Gerganov, first published in 2022[^1]. It provides the primitives — n-dimensional tensors, a static compute graph, quantized data types, and a set of hardware backends — that make it possible to run neural-network inference on commodity hardware without CUDA, without Python, and without a heavyweight runtime. It is the compute core that llama.cpp and whisper.cpp are built on, and by extension it powers a large share of the local-LLM ecosystem (Ollama, LM Studio, llamafile, and many downstream apps embed it).

The defining design choice is **no third-party dependencies and no allocations during inference**[^2]. A model's entire memory footprint is planned up front into a preallocated context; the compute graph is built once and executed, rather than eagerly evaluating operations as PyTorch does. This makes ggml embeddable in almost anything (a phone app, a browser via WASM, a microcontroller-class device) and predictable in memory use, at the cost of ergonomics: you build graphs by hand in C, and the API is comparatively raw.

The defining *tension* is organizational, not technical. ggml is nominally the upstream library, but most active development happens inside the [llama.cpp](https://github.com/ggml-org/llama.cpp) repository, where the same `ggml.c`/`ggml.h` sources live and evolve fastest[^2]. This standalone repo is periodically synced from there. Treat llama.cpp as the source of truth for the newest backends and ops, and this repo as the clean, example-oriented distribution of the core.

## Getting Started

```bash
git clone https://github.com/ggml-org/ggml
cd ggml
mkdir build && cd build
cmake ..
cmake --build . --config Release -j 8
```

Run one of the bundled examples (GPT-2 inference on CPU):

```bash
../examples/gpt-2/download-ggml-model.sh 117M
./bin/gpt-2-backend -m models/gpt-2-117M/ggml-model.bin -p "This is an example"
```

A minimal graph in C — allocate a context, build `a * b`, evaluate:

```c
#include "ggml.h"
#include "ggml-cpu.h"

struct ggml_init_params params = { .mem_size = 16*1024*1024, .no_alloc = false };
struct ggml_context * ctx = ggml_init(params);

struct ggml_tensor * a = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4);
struct ggml_tensor * b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4);
struct ggml_tensor * c = ggml_mul(ctx, a, b);

struct ggml_cgraph * gf = ggml_new_graph(ctx);
ggml_build_forward_expand(gf, c);
// set a/b data, then: ggml_graph_compute_with_ctx(ctx, gf, /*n_threads*/4);
ggml_free(ctx);
```

## Architecture / How It Works

ggml separates **graph definition** from **graph execution**:

1. **Context** — a bump allocator (`ggml_context`) holds all tensor metadata and, optionally, tensor data. You size it up front; ggml does not malloc during compute.
2. **Tensors** — up to 4 dimensions, described by shape (`ne`) and strides (`nb`). Views, transposes, and reshapes are metadata-only where possible.
3. **Compute graph** (`ggml_cgraph`) — operations are recorded lazily into a DAG of nodes. `ggml_build_forward_expand` walks the graph; a backend scheduler then executes it. Autodiff exists for a subset of ops, but training is a secondary use case.
4. **Backends** — the CPU backend uses hand-written SIMD kernels (AVX/AVX2/AVX-512, ARM NEON, and increasingly dot-product/matmul extensions). GPU and accelerator backends include CUDA, Metal, Vulkan, SYCL, HIP/ROCm, CANN (Ascend), and OpenCL. `ggml-backend` abstracts scheduling and tensor offload across them.

**Quantization** is the reason ggml runs large models on small machines. Weights are stored in block-quantized formats (Q4_0, Q4_K, Q5_K, Q6_K, Q8_0 and the more recent I-quants and importance-matrix variants), typically 2–8 bits per weight, dequantized on the fly inside the matmul kernel[^3]. This trades accuracy for a multiple-x reduction in memory and bandwidth, and bandwidth is the binding constraint for LLM decode.

**GGUF** is ggml's model container format, introduced in 2023 to replace the older ad-hoc GGML/GGJT formats[^4]. It is a single-file format holding tensors plus arbitrary key/value metadata (architecture, tokenizer, chat template, quantization type), designed to be extensible without breaking older loaders. Nearly every local-inference tool now consumes GGUF.

## Production Notes

- **Pin to llama.cpp, not this repo, for serious inference.** The standalone ggml repo trails llama.cpp on new operators, quant types, and backend fixes. If you need a specific model architecture or the latest Metal/CUDA kernel, vendor ggml from llama.cpp's tree.
- **No stable API or semantic versioning.** ggml has no released version numbers; it ships as source you compile. Function signatures, enum values, and struct layouts change between commits. Vendoring pins a commit; upgrading is a manual diff, not a package bump.
- **GGUF is forward-ish, not backward, compatible.** New quant types and metadata keys mean a model quantized with a recent tool may fail to load in an older binary. The failure mode is usually a clear error, but CI should test the exact loader/model pairing you ship.
- **Quantization quality is not uniform.** Legacy `Q4_0` is faster to produce but lower quality than K-quants at the same bit budget; I-quants need an importance matrix (imatrix) computed from calibration data. Blindly picking a quant by size alone can cost measurable perplexity.
- **Memory planning is your responsibility.** Undersize the context or KV-cache and you get allocation failures or truncated context, not graceful degradation. Context length, batch size, and KV-cache dtype all multiply into the up-front reservation.
- **Backend availability is a build-time decision.** Each backend is a compile flag (`-DGGML_CUDA=ON`, `-DGGML_METAL=ON`, `-DGGML_VULKAN=ON`, …). A binary built without a backend cannot use that hardware at runtime; distributing broadly means building a matrix of variants.
- **Numerics differ across backends.** The same model can produce slightly different logits on CPU vs CUDA vs Metal due to kernel and reduction-order differences. Don't assume bit-identical output across hardware.

## When to Use / When Not

**Use when:**
- You need to run LLM/whisper/embedding inference locally, on CPU or consumer GPU, with tight and predictable memory.
- You want to embed inference into a C/C++/mobile/WASM application with zero Python and no external runtime.
- You need aggressive quantization (2–8 bit) to fit a model that would not otherwise fit.

**Avoid when:**
- You are training models at scale — use a full framework with mature autograd and distributed support.
- You want a stable, semver-versioned library API you can depend on without vendoring.
- You need broad, day-one model-architecture coverage from a high-level Python API — a served-inference stack fits better.
- Your deployment is cloud GPUs where a batched server (vLLM, TensorRT-LLM) will beat single-request local inference on throughput.

## Alternatives

- ggml-org/llama.cpp — where ggml is actually developed; use it directly when you want the newest ops, backends, and a batteries-included CLI/server.
- pytorch/pytorch — use instead when you need training, autograd, and the broad research ecosystem rather than embeddable inference.
- microsoft/onnxruntime — use instead when you have ONNX models and want a cross-platform, versioned inference runtime with vendor support.
- ml-explore/mlx — use instead when you target Apple silicon specifically and want a NumPy-like Python API with unified memory.
- ggerganov/whisper.cpp — use instead for speech-to-text specifically; it is the ggml-based Whisper implementation.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo created | 2022-09 | ggml published as a standalone tensor library[^1]. |
| whisper.cpp | 2022-09 | First high-profile ggml consumer — CPU Whisper inference. |
| llama.cpp | 2023-03 | LLaMA inference on CPU; drives most ggml development thereafter[^2]. |
| ggml.ai | 2023-06 | Company formed around the project (pre-seed backing)[^5]. |
| GGUF format | 2023-08 | New extensible model container replaces GGML/GGJT[^4]. |
| ggml-org | 2024–2025 | Project moved to the `ggml-org` GitHub organization. |
| Ongoing | 2026-07 | Active development; last push 2026-07-10. |

Numbers as of this writing: ~15,000 stars, ~1,700 forks, MIT-licensed, last pushed 2026-07-10 — actively maintained, though the center of gravity remains the llama.cpp repository.

## References

[^1]: ggml — Tensor library for machine learning. https://github.com/ggml-org/ggml
[^2]: llama.cpp repository, where the ggml core is co-developed. https://github.com/ggml-org/llama.cpp
[^3]: "Introduction to ggml" — Hugging Face blog, on tensors, graphs, and quantization. https://huggingface.co/blog/introduction-to-ggml
[^4]: GGUF file format specification. https://github.com/ggml-org/ggml/blob/master/docs/gguf.md
[^5]: ggml.ai — project/company site. https://ggml.ai

## Tags

c, cpp, tensor-library, machine-learning, llm-inference, quantization, gguf, on-device, whisper, llama-cpp, automatic-differentiation, simd
