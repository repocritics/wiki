# tracel-ai/burn

> A Rust deep learning framework where the same code trains a model and runs it in production — across CPU, CUDA, ROCm, Metal, Vulkan, WebGPU, and the browser.

[GitHub repo](https://github.com/tracel-ai/burn) ·
[Official website](https://burn.dev) ·
[License: MIT OR Apache-2.0](https://github.com/tracel-ai/burn#license)

## Overview

Burn is a tensor library and deep learning framework written in Rust, first published in 2022[^1]. Its central bet is that training and inference should not live in separate worlds. The common industry pattern — train in Python, export to ONNX or a serving engine (vLLM, ONNX Runtime, TensorRT), then debug the lossy export — is exactly what Burn is designed to avoid. Because model code is generic over a `Backend` trait, the code you train with is the code you deploy, and the choice of hardware is a type parameter rather than a rewrite[^2].

The defining architectural idea is the **backend as a composable decorator**. A base backend (say CUDA or the pure-Rust CPU backend) can be wrapped with `Autodiff` to gain backpropagation, with `Fusion` to gain automatic kernel fusion, and with a remote decorator to ship tensor ops over the network. These compose: `Autodiff<Fusion<Cuda>>` is a valid, meaningful type. This is unusual — most frameworks bake autodiff and the execution engine together — and it is the source of both Burn's flexibility and its steeper conceptual overhead.

The tradeoff to weigh is ecosystem maturity against portability and safety. Burn is still pre-1.0 and moves fast; its pretrained-model catalog and third-party tutorials are a fraction of PyTorch's. In exchange you get Rust's memory safety, single-binary deployment, `no_std` embedded targets, and one codebase that runs from a microcontroller to a GPU cluster. Whether that trade pays off depends heavily on how much you value deployment portability over Python's incumbent gravity.

## Getting Started

```bash
cargo add burn --features wgpu       # cross-platform GPU via WebGPU
# or: --features cuda | rocm | metal | vulkan | ndarray | tch
```

```rust
use burn::backend::{Autodiff, Wgpu};
use burn::tensor::{Distribution, Tensor};

fn main() {
    // Autodiff wraps a base backend to enable .backward().
    type Backend = Autodiff<Wgpu>;
    let device = Default::default();

    let x: Tensor<Backend, 2> = Tensor::random([32, 32], Distribution::Default, &device);
    let y: Tensor<Backend, 2> =
        Tensor::random([32, 32], Distribution::Default, &device).require_grad();

    let z = (x.clone() + y.clone()).matmul(x).exp();

    let grads = z.backward();
    let y_grad = y.grad(&grads).unwrap();
    println!("{y_grad}");
}
```

Calling `.backward()` is only available on an `Autodiff` backend, so an inference-only backend makes the mistake unrepresentable at compile time.

## Architecture / How It Works

**Generic over `Backend`.** Nearly all of Burn — layers, optimizers, the training loop — is written against the `Backend` trait rather than a concrete tensor type. Swapping hardware means changing one type alias. This is why decorators work: `Autodiff`, `Fusion`, and the remote client/server are themselves backends that wrap an inner backend[^2].

**CubeCL is the compute layer.** The first-party accelerated backends (CUDA, ROCm, Metal, Vulkan, WebGPU, and a CubeCL CPU backend) are built on CubeCL[^3], a sibling project that lets you write GPU kernels once in Rust and compile them to each platform's shading/compute language. CubeCL is usable standalone. The external backends — LibTorch (`tch`) bindings and the pure-Rust `ndarray`/Flex CPU backend — do not go through CubeCL and compose with `Autodiff` only, not `Fusion`.

**JIT kernel fusion.** Rather than eagerly dispatching each tensor op, Burn's fused backends record a stream of operations and JIT-compile them into fewer, larger kernels. This recovers much of the throughput that dynamic (define-by-run) graphs normally cost. All first-party accelerated backends enable `Fusion` by default.

**Model interop.** `burn-onnx` imports ONNX models by generating native Burn/Rust source, so the imported model runs on any backend and benefits from fusion — but only a limited subset of ONNX operators is supported, and coverage is the usual friction point. `burn-store` loads weights from PyTorch `.pt` and Safetensors formats into a Burn-defined module.

**Training and embedded.** A Ratatui-based terminal dashboard shows live train/validation metrics and lets you break out of the loop without corrupting an in-flight checkpoint. Core components support `no_std` for bare-metal targets, though as of this writing only the Flex CPU backend runs in a `no_std` environment.

## Production Notes

**Compile times are the real cost.** Rust's edit-compile-run loop was historically the reason researchers avoided it. Burn is engineered around incremental compilation and advertises sub-5-second rebuilds on model changes even in release mode, but *cold* builds pulling in CUDA/wgpu toolchains are slow, and CI caching (`sccache`, warm target dirs) matters more here than in a Python project.

**The wgpu recursion-limit footgun.** With any `wgpu`-family backend you may hit a compiler error about recursive type evaluation, caused by deep type nesting in the `wgpu` dependency chain. The fix is to add `#![recursion_limit = "256"]` at the top of `main.rs`/`lib.rs`; the default of 128 is just below what the type graph needs. This bites nearly every new WebGPU user.

**Backend capability is not uniform.** The support matrix is real and asymmetric: CUDA is Nvidia-only, ROCm is AMD-only, Metal is Apple-only, and WebGPU/Vulkan are the portable-but-sometimes-slower options. LibTorch and the CubeCL CPU/Flex backends have different feature sets (e.g. Flex is the only `no_std`/browser-CPU option). Pick a backend before you assume an operator or a decorator (`Fusion`) is available.

**Pre-1.0 churn.** Burn ships frequent 0.x releases and does break APIs between them. Pin exact versions, read release notes before upgrading, and expect the tensor and module APIs to shift more than a 1.0 framework would. The ONNX operator set, in particular, expands release to release.

**Ecosystem gaps.** There is far less off-the-shelf material than PyTorch: fewer pretrained checkpoints (the `tracel-ai/models` repo is the curated starting point), fewer Stack Overflow answers, and thinner coverage for exotic layers. Budget time to implement things you would `pip install` elsewhere.

## When to Use / When Not

**Use when:**
- You want one codebase to train and deploy without an ONNX/TorchScript export step.
- Deployment portability matters — the same model must run on GPU servers, WebAssembly, and embedded/`no_std` targets.
- You want Rust's memory safety and single-binary distribution for your ML service.
- On-device personalization, federated learning, or browser inference is a first-class requirement.

**Avoid when:**
- You need the largest ecosystem of pretrained models and tutorials today — that is PyTorch.
- Your team is Python-first and the Rust learning curve outweighs the deployment payoff.
- You are mainly running existing LLMs rather than training custom architectures — a focused inference engine is lighter.
- You need long-term API stability now; a pre-1.0 framework will move under you.

## Alternatives

- huggingface/candle — minimalist Rust ML framework focused on LLM inference; use it when you mostly want to *run* existing models rather than train novel architectures across many backends.
- pytorch/pytorch — the incumbent; use it when ecosystem size, pretrained weights, and research reproducibility matter more than deployment portability or Rust safety.
- tracel-ai/cubecl — Burn's own GPU compute layer; use it directly when you want to write portable Rust GPU kernels without the full deep-learning framework on top.
- tinygrad/tinygrad — tiny, lazy, multi-backend framework; use it when you want a small hackable codebase and are comfortable on the bleeding edge.
- sonos/tract — pure-Rust ONNX/NNEF inference engine; use it when you only need to serve an already-trained model in Rust and don't need training or GPU fusion.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2022-07 | Repository created; project begins as a Rust tensor + DL framework[^1]. |
| 0.x | 2023 | Burn Book docs, autodiff/fusion decorator model, wgpu backend mature. |
| 0.x | 2024 | CubeCL extracted as the standalone GPU compute layer behind native CUDA/Metal/ROCm/Vulkan backends[^3]. |
| 0.x | 2026 | Actively developed; last push 2026-07-14. Still pre-1.0. |

## References

[^1]: tracel-ai/burn repository, created 2022-07-18. https://github.com/tracel-ai/burn
[^2]: Burn README — backend trait, Autodiff/Fusion/Remote decorators, and the "same code for training and inference" design. https://github.com/tracel-ai/burn#backend
[^3]: CubeCL — GPU compute language and compiler powering Burn's accelerated backends. https://github.com/tracel-ai/cubecl

## Tags

rust, deep-learning, machine-learning, tensor, autodiff, kernel-fusion, gpu, webgpu, cuda, onnx, cross-platform, neural-network
