# huggingface/candle

> A minimalist ML framework for Rust, built by Hugging Face to make serverless model inference possible without Python.

[GitHub repo](https://github.com/huggingface/candle) ·
[Documentation](https://huggingface.github.io/candle/) ·
[License: Apache-2.0](https://github.com/huggingface/candle/blob/main/LICENSE-APACHE)

## Overview

Candle is a tensor and neural-network library for Rust, developed by Hugging Face and first published to crates.io in 2023[^1]. Its stated goal is narrow and deliberate: make it practical to deploy model inference as small, fast, Python-free binaries — the kind of thing that starts quickly on a serverless platform where a full PyTorch image would be slow to pull and slow to boot[^2]. It is dual-licensed MIT / Apache-2.0.

The API is designed to look and feel like PyTorch — `Tensor::randn`, `.matmul`, `.reshape`, `.to_dtype` — so that porting a model is mostly mechanical. The defining difference is Rust's error model: nearly every tensor operation returns a `Result`, so real Candle code is a chain of `?` operators, and shape/dtype/device mismatches surface as runtime `Err` values rather than compile-time type errors (unlike `dfdx`, which encodes shapes in the type system). That is the central tradeoff — Candle keeps the ergonomics of a dynamic framework and does not try to make the type checker prove your tensor shapes.

Candle is inference-first. It supports training and has an autograd engine, but the ecosystem energy — the large `candle-transformers` model zoo, the quantized/GGUF support, the WASM demos — is overwhelmingly about running published models efficiently, not about being a research training framework. It sits alongside two other Hugging Face Rust crates, `safetensors` and `tokenizers`, and is the compute layer many Rust-native inference projects build on.

## Getting Started

Add the core crate to a Cargo project:

```bash
cargo add candle-core
# GPU builds: cargo add candle-core --features cuda   (or --features metal on macOS)
```

A minimal matrix multiply on CPU:

```rust
use candle_core::{Device, Tensor};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let device = Device::Cpu;

    let a = Tensor::randn(0f32, 1., (2, 3), &device)?;
    let b = Tensor::randn(0f32, 1., (3, 4), &device)?;

    let c = a.matmul(&b)?;   // Tensor[[2, 4], f32]
    println!("{c}");
    Ok(())
}
```

Moving the same computation to a GPU is a one-line device change:

```rust
let device = Device::new_cuda(0)?;   // or Device::new_metal(0)? on Apple silicon
```

Higher-level model code uses `candle-nn` (layers, optimizers) and a `VarBuilder` to load weights from safetensors; the `candle-transformers` crate ships ready-made implementations of many published architectures.

## Architecture / How It Works

Candle is a Cargo workspace of focused crates:

- **candle-core** — the `Tensor` type, `DType`, `Device`, and the core ops. This is the layer that abstracts over backends.
- **candle-nn** — building blocks for real models: linear layers, layernorm, embeddings, optimizers, and `VarBuilder`/`VarMap` for weight loading.
- **candle-transformers** — concrete implementations of published models (LLaMA, Mistral, Mixtral, Whisper, Stable Diffusion, BERT, T5, YOLO, SAM, and many more). Community-contributed and the reason most people reach for Candle.
- **candle-kernels** / **candle-metal-kernels** — the hand-written CUDA and Metal kernels.
- **candle-flash-attn** — a FlashAttention v2 binding (CUDA only).
- **candle-onnx** — evaluates ONNX graphs.
- **candle-datasets**, **candle-examples**, **candle-wasm-examples** — data loaders, runnable examples, browser demos.

**Backends.** `Device` dispatches ops to one of several backends: an optimized CPU backend (with optional Intel MKL on x86 or Apple Accelerate on macOS), a CUDA backend (with optional cuDNN and multi-GPU distribution via NCCL), a Metal backend for Apple silicon, and a WASM target for running in the browser[^3]. Backends are gated by Cargo features, so a CPU-only build pulls in none of the GPU toolchain.

**Weight formats.** Candle loads safetensors, `.npz`, GGML/GGUF, and PyTorch `.pth` files. Quantization reuses the same quantized tensor types as `llama.cpp`, which is what makes GGUF quantized LLMs (Q4_0, Q4_K, etc.) run directly.

**Autograd.** Tensors track their computation graph; calling `.backward()` produces gradients that `candle-nn` optimizers consume. This is functional but intentionally smaller in scope than PyTorch's; the library's own framing is that it exists primarily to enable inference, with training as a supported secondary use.

## Production Notes

**GPU build cost.** Enabling `--features cuda` compiles the custom CUDA kernels, which adds real time to a clean build and requires a working CUDA toolkit. `--features cudnn` and `candle-flash-attn` add more. Budget for this in CI and Docker layers; cache the compiled artifacts.

**Metal is younger than CUDA.** The Metal backend is real and actively used, but it has historically lagged CUDA in op coverage and has had correctness/perf issues fixed over time. If you target Apple silicon, validate the specific model you need rather than assuming parity; some ops may be missing or slower.

**Errors are runtime, not compile-time.** Because shapes and dtypes are not in the type system, a wrong `reshape`, a device mismatch, or an F16/F32 mixup fails at execution as an `Err`, often deep in a model forward pass. Good error messages help, but there is no borrow-checker-style guarantee that your tensor plumbing is correct before you run it.

**Model coverage is community-driven and can lag.** `candle-transformers` is broad but not exhaustive, and a newly released architecture may not have a Candle implementation, or may have one that only covers the common path. Check that the exact model, quantization, and features you need are implemented before committing — porting a new architecture is nontrivial work.

**Pre-1.0 API churn.** Candle is still on 0.x. Minor-version bumps have carried breaking API changes; pin your version and read release notes before upgrading, and expect `candle-transformers` to move faster than a stable framework would.

**WASM constraints.** In-browser inference works and is a headline feature, but you inherit WASM's limits: large model weights to download, single-threaded execution unless you configure threads, and memory ceilings. It suits small/quantized models, not 70B LLMs.

**Linker errors with MKL/Accelerate.** Building with the `mkl` or `accelerate` features can produce "undefined reference to `sgemm_`" linker failures; the documented fix is to add `extern crate intel_mkl_src;` (or `extern crate accelerate_src;`) at the top of your binary crate[^4].

## When to Use / When Not

**Use when:**
- You want to serve model inference as a small, fast-starting Rust binary without a Python runtime.
- You are already in the Rust/Hugging Face ecosystem (safetensors, tokenizers) and want a matching compute layer.
- You need in-browser (WASM) inference of small or quantized models.
- You want to run GGUF-quantized LLMs from Rust with a `llama.cpp`-compatible quantization scheme.

**Avoid when:**
- You are doing research-scale training and need PyTorch's full autograd, ecosystem, and tooling.
- You need a specific brand-new architecture that no one has ported yet, and you can't afford to implement it.
- You depend on compile-time shape checking to catch tensor bugs — Candle checks at runtime.
- Your team wants API stability and rare breaking changes; a 0.x framework is the wrong fit.

## Alternatives

- tracel-ai/burn — general-purpose Rust deep-learning framework with pluggable backends (including a Candle backend); use it when you want backend flexibility and a training-first design rather than inference-first minimalism.
- LaurentMazare/tch-rs — Rust bindings to libtorch; use when you want PyTorch's full capability and can accept pulling the entire Torch library into your runtime. (Its main author also contributes to Candle.)
- coreylowman/dfdx — Rust tensors with shapes encoded in the type system; use when you want the compiler to reject shape mismatches, and can tolerate nightly features.
- ggml-org/llama.cpp — C/C++ inference engine for quantized LLMs; use when you want the most mature quantized-LLM runtime and don't need Rust.
- pytorch/pytorch — the Python reference framework; use when you are training or researching and inference-binary size is not a concern.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2023-06-19 | Public development begins under huggingface/candle[^5]. |
| 0.1.x | 2023-08 | First crates.io releases of candle-core / candle-nn[^1]. |
| 0.x (ongoing) | 2023–2026 | Metal backend, GGUF/quantized support, growing candle-transformers zoo, ONNX eval; still pre-1.0 with breaking minor releases. |

## References

[^1]: candle-core on crates.io. https://crates.io/crates/candle-core
[^2]: Candle README, "Why should I use Candle?" — core goal is serverless inference and removing Python from production workloads. https://github.com/huggingface/candle#faq
[^3]: Candle README, "Features" — CPU (MKL/Accelerate), CUDA (cuDNN, NCCL multi-GPU), and WASM backends. https://github.com/huggingface/candle#features
[^4]: Candle README, "Common Errors — Missing symbols when compiling with the mkl feature." https://github.com/huggingface/candle#common-errors
[^5]: GitHub repository metadata, huggingface/candle. https://github.com/huggingface/candle

## Tags

rust, machine-learning, ml-framework, inference, tensor, llm, wasm, cuda, quantization, serverless, huggingface, pytorch-alternative
