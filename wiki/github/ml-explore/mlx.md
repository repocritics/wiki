# ml-explore/mlx

> A NumPy-shaped array framework built by Apple's ML research group around unified memory on Apple silicon.

[GitHub repo](https://github.com/ml-explore/mlx) ·
[Official docs](https://ml-explore.github.io/mlx/) ·
[License: MIT](https://github.com/ml-explore/mlx/blob/main/LICENSE)

## Overview

MLX is an array and machine-learning framework, first released by Apple's machine learning research team in December 2023[^1]. Its Python API deliberately tracks NumPy, while the higher-level `mlx.nn` and `mlx.optimizers` packages track PyTorch, so most of the surface is familiar on day one. Underneath, it borrows its execution model — composable `grad` / `vmap` / `compile` transformations over a lazily-built graph — from JAX[^2].

The framework's defining bet is Apple silicon's *unified memory*. On an M-series chip the CPU and GPU share one physical pool of RAM, and MLX exposes that directly: arrays live in shared memory and operations dispatch to CPU or GPU with no host-to-device copy. That single design choice is what lets a consumer MacBook run a 30B-parameter quantized model — the "GPU memory" is simply all of system RAM — and it is also what pins the framework's original identity to one hardware family.

The tension worth stating up front: MLX is genuinely pleasant and fast *on a Mac*, but for its first two years the GPU path meant Metal and only Metal. A CUDA backend for Linux/NVIDIA has since landed[^3], broadening reach, but the ecosystem, the tuned kernels, and the community weight still assume Apple silicon. It is a research-and-local-inference framework that happens to be excellent at running LLMs on your laptop, not a drop-in replacement for a CUDA training stack.

## Getting Started

```bash
pip install mlx            # macOS, Apple silicon (Metal GPU)
pip install mlx[cuda]      # Linux + NVIDIA CUDA backend
pip install mlx[cpu]       # Linux, CPU-only
```

```python
import mlx.core as mx

def f(x):
    return (x ** 2).sum()

x = mx.array([1.0, 2.0, 3.0])

# Composable transforms: gradient of f, then compiled
grad_f = mx.compile(mx.grad(f))

g = grad_f(x)     # lazy graph built here
mx.eval(g)        # nothing actually computes until eval (or printing)
print(g)          # array([2, 4, 6], dtype=float32)
```

The `mx.eval` call is the part newcomers miss: MLX is lazy, so arrays are placeholders in a graph until something forces materialization. For LLM inference and finetuning, most people use the companion `mlx-lm` package rather than writing model code by hand.

## Architecture / How It Works

MLX is a C++ core with thin language bindings; the Python, [C](https://github.com/ml-explore/mlx-c), and [Swift](https://github.com/ml-explore/mlx-swift) front-ends mirror the same operations[^1]. Four ideas define its behavior:

1. **Unified memory.** Arrays are allocated once in shared memory. There is no `.to("cuda")` / `.cpu()` dance and no explicit device transfer — you pass a `device=` at op time, and the same buffer is read by whichever processor runs the kernel. This removes an entire class of copy bugs and is the framework's central differentiator.
2. **Lazy evaluation.** Operations record nodes in a graph rather than computing immediately. Work happens at `mx.eval`, when you print, or when a value is otherwise needed. This lets MLX fuse and schedule, but means a forgotten `eval` can silently grow an unbounded graph.
3. **Dynamic graph construction.** Graphs are rebuilt each call, so changing input shapes does *not* trigger a recompile the way a statically-traced framework would. Debugging is closer to eager PyTorch than to a traced JAX program.
4. **Composable transforms.** `grad`, `value_and_grad`, `vmap`, and `compile` are functions that take functions and return functions, and they nest. `compile` traces and fuses a graph into a single optimized kernel launch for the hot path.

The GPU backend is Metal on Apple silicon, with hand-written Metal kernels for the primitive ops. The newer CUDA backend[^3] targets NVIDIA on Linux and is the framework's first step off Apple-only hardware; treat it as maturing rather than at parity with the Metal path. A separate `mlx.distributed` module supports multi-device and multi-machine training over MPI and a built-in ring backend.

## Production Notes

**"Production" here usually means local inference, not a training cluster.** The dominant real-world use is running and finetuning LLMs on Macs via `mlx-lm` — 4-bit and 8-bit quantization are first-class, and throughput on M-series GPUs is competitive with llama.cpp's Metal path for many models.

- **Memory is the ceiling, and it's shared.** Because model weights, activations, and everything else draw from the same unified pool as the OS and your other apps, the practical model-size limit is total RAM minus headroom, not a separate VRAM budget. A 128 GB Mac Studio can hold models a 24 GB discrete GPU cannot; an 8 GB MacBook Air hits the wall fast. Watch for the OS beginning to swap.
- **Forgetting `eval` is the classic footgun.** In a generation loop, failing to evaluate per step lets the lazy graph accumulate across iterations and blows up memory. `mlx-lm` handles this internally; hand-written loops must not.
- **No CUDA-ecosystem shortcuts on the Metal path.** You cannot drop in FlashAttention CUDA kernels, `bitsandbytes`, Triton, or vendor libraries that assume NVIDIA. MLX ships its own kernels; if an operation isn't implemented or tuned, there is no fallback to a mature third-party CUDA library on Apple hardware.
- **Pre-1.0, moving fast.** MLX is still on `0.x` versioning[^4]. The API is stable in spirit but minor releases are frequent and occasionally adjust behavior; pin versions in anything you depend on.
- **Not a broad hardware story.** No AMD GPU, no Intel GPU, no TPU. GPU acceleration means Apple silicon (Metal) or the newer NVIDIA/CUDA backend — nothing in between. Intel Macs are CPU-only.
- **Small but real ops gap vs. PyTorch.** Most common layers and ops exist, but exotic or very new operators may be missing, and you may occasionally need to implement a custom Metal kernel or fall back to a NumPy-style composition.

## When to Use / When Not

**Use when:**
- You're running or finetuning models locally on Apple silicon and want native performance without CUDA.
- You want a NumPy/PyTorch-shaped API with JAX-style `grad`/`vmap`/`compile` transforms.
- You value the unified-memory model — large models on consumer hardware, no device-copy bookkeeping.
- You're doing ML research on a Mac and want to prototype and iterate quickly.

**Avoid when:**
- Your training or serving target is an NVIDIA cluster: PyTorch + CUDA has vastly more tooling, tuned kernels, and community weight.
- You need broad hardware portability (AMD, TPU, mobile NPUs) from one codebase.
- You depend on the CUDA ecosystem (Triton, FlashAttention kernels, `bitsandbytes`, DeepSpeed).
- You need a locked-down, rarely-changing API for long-lived production systems — MLX is pre-1.0 and iterates.

## Alternatives

- google/jax — the closest conceptual sibling; same functional-transform model (`grad`/`vmap`/`jit`) but XLA-backed on GPU/TPU. Use JAX when you need TPUs or NVIDIA scale rather than Apple-local performance.
- pytorch/pytorch — has an MPS backend for Apple silicon; use it when you want the largest ecosystem and are willing to accept a heavier, less Apple-tuned path.
- ggml-org/llama.cpp — if you only need LLM inference (GGUF) on a Mac with a Metal backend, this is leaner and has no Python dependency.
- tinygrad/tinygrad — a small, portable framework spanning many backends; use it when cross-hardware minimalism matters more than Apple-native speed.
- ml-explore/mlx-swift — the official Swift binding of this same core; use it to embed MLX in native macOS/iOS apps.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Public release | 2023-12 | Initial open-source release by Apple ML research; repo created 2023-11-28[^1]. |
| Swift / C bindings | 2024 | `mlx-swift` and `mlx-c` front-ends over the shared C++ core. |
| `mlx-lm` ecosystem | 2024 | Dedicated package for LLM inference, quantization, and finetuning on Mac. |
| CUDA backend | 2025 | Linux/NVIDIA GPU support added, first step beyond Apple-only hardware[^3]. |

*(Still on `0.x` versioning as of mid-2026[^4]; releases are frequent, so consult the changelog for exact per-version behavior.)*

## References

[^1]: MLX README and repository, Apple ml-explore. https://github.com/ml-explore/mlx
[^2]: MLX documentation — the design cites NumPy, PyTorch, Jax, and ArrayFire as inspirations. https://ml-explore.github.io/mlx/
[^3]: MLX installation docs — `pip install mlx[cuda]` for the Linux/NVIDIA CUDA backend; `mlx[cpu]` for CPU-only Linux. https://ml-explore.github.io/mlx/build/html/install.html
[^4]: MLX on PyPI — release versioning. https://pypi.org/project/mlx/

## Tags

apple-silicon, machine-learning, array-framework, metal, unified-memory, python, cpp, llm-inference, autograd, on-device
