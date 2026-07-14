# pytorch/pytorch

> Tensors and dynamic neural networks in Python with strong GPU acceleration — the define-by-run framework that became the default for ML research.

[GitHub repo](https://github.com/pytorch/pytorch) ·
[Official website](https://pytorch.org) ·
[License: BSD-3-Clause](https://github.com/pytorch/pytorch/blob/main/LICENSE)

## Overview

PyTorch is a tensor and automatic-differentiation library with first-class GPU support, used primarily to define, train, and deploy neural networks. It grew out of the Lua-based Torch7 project; the Python rewrite was led by Adam Paszke, Sam Gross, Soumith Chintala, and Adam Lerer at Facebook AI Research, with the first public release in early 2017[^1]. Its defining design choice was **define-by-run** (dynamic computation graphs): the graph is built implicitly as Python executes, rather than declared up front and then run in a separate session. At the time this was the sharp differentiator against TensorFlow 1.x's static graphs, and it is the main reason PyTorch won the research community — models are ordinary Python, debuggable with `pdb` and `print`.

The central tension of PyTorch's history is **eager flexibility vs. compiled performance**. Define-by-run is easy to write but hard to optimize, because there is no whole-graph view to fuse kernels or plan memory. For years the answer was TorchScript (`torch.jit`), a separate graph-capture path that few teams adopted cleanly. PyTorch 2.0 (2023) reframed the whole problem with `torch.compile`, a JIT that captures graphs from live Python via bytecode analysis and falls back to eager on anything it cannot trace[^2]. That single feature — "compile when you can, stay eager when you can't" — is the modern shape of the framework.

PyTorch is maintained under the PyTorch Foundation, part of the Linux Foundation since 2022[^3], though Meta remains the dominant contributor. It is the substrate for most of the current LLM ecosystem (Hugging Face Transformers, vLLM, most training stacks target it) and its issue tracker reflects a very large, actively developed codebase spanning Python, C++, and CUDA.

## Getting Started

Install the build matching your accelerator (the index URL selects the CUDA/ROCm/CPU variant):

```bash
# CUDA 12.x build (see pytorch.org for the current selector)
pip install torch --index-url https://download.pytorch.org/whl/cu124
# CPU-only
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

A minimal training step showing autograd — the graph is built as the code runs:

```python
import torch
import torch.nn as nn

device = "cuda" if torch.cuda.is_available() else "cpu"

model = nn.Sequential(nn.Linear(784, 128), nn.ReLU(), nn.Linear(128, 10)).to(device)
opt = torch.optim.AdamW(model.parameters(), lr=1e-3)
loss_fn = nn.CrossEntropyLoss()

x = torch.randn(64, 784, device=device)
y = torch.randint(0, 10, (64,), device=device)

logits = model(x)              # forward: graph recorded on the fly
loss = loss_fn(logits, y)
opt.zero_grad()
loss.backward()                # reverse-mode autodiff over the recorded graph
opt.step()
```

Opting into compilation is a single wrapper; the API surface stays eager:

```python
model = torch.compile(model)   # captures graphs, falls back to eager where it can't
```

## Architecture / How It Works

PyTorch is a Python front-end over a large C++ core:

- **`torch.Tensor` / autograd** — every op on a tensor with `requires_grad=True` records a node in a dynamic backward graph. `loss.backward()` walks that graph in reverse (reverse-mode autodiff), accumulating gradients into `.grad`. The graph is rebuilt every forward pass, which is what makes control flow (loops, conditionals, variable-length inputs) trivial to express.
- **ATen** — the C++ tensor library that implements the thousands of operators (`add`, `matmul`, `conv2d`, …), dispatching per-device. **c10** is the lower core (device, dtype, `TensorImpl`). The **dispatcher** routes each op call to the right backend kernel (CPU, CUDA, autograd wrapper, etc.) via a key-based table — this is how autograd, AMP, and functorch-style transforms layer on without touching kernel code.
- **Backends** — CUDA is the primary target; ROCm (AMD), MPS (Apple Silicon Metal), and XPU (Intel) are also supported, with varying operator coverage and maturity. CPU kernels use oneDNN/MKL where available.
- **`torch.compile` (2.x)** — three cooperating pieces: **TorchDynamo** hooks CPython frame evaluation to capture FX graphs from real bytecode, breaking the graph ("graph breaks") wherever it hits untraceable Python and resuming in eager; **AOTAutograd** traces the backward pass ahead of time; **TorchInductor** lowers the captured graph to fused kernels (Triton on GPU, C++/OpenMP on CPU)[^2]. The design goal was to get compiled speedups without asking users to rewrite eager code.
- **Legacy capture paths** — TorchScript (`torch.jit.script`/`trace`) predates `torch.compile` and is now in maintenance mode; `torch.export` is the newer path for producing a serializable, ahead-of-time graph for deployment.
- **Distributed** — `torch.distributed` provides collectives (NCCL/Gloo). `DistributedDataParallel` (DDP) replicates the model per GPU; **FSDP** (Fully Sharded Data Parallel) shards parameters, gradients, and optimizer state across ranks for models too large to replicate.

The coupling that matters most operationally is **PyTorch ↔ CUDA ↔ driver ↔ GPU architecture**. A given `torch` wheel is built against a specific CUDA toolkit version and a set of compute capabilities; mismatches surface as install-time or first-`.cuda()`-call failures rather than clean errors.

## Production Notes

**CUDA / version matrix is the top footgun.** Each PyTorch release supports a bounded set of CUDA versions, and each CUDA version supports a bounded set of driver versions and GPU architectures. Installing the wrong wheel yields `no kernel image is available for execution on the device` (architecture mismatch) or silent CPU fallback. Pin the wheel via the correct `--index-url` and verify with `torch.version.cuda` and `torch.cuda.get_device_capability()`.

**GPU memory is a caching allocator, not `nvidia-smi` truth.** PyTorch keeps freed blocks in a caching allocator, so `nvidia-smi` shows memory that is free for PyTorch but not returned to the OS. Fragmentation causes `CUDA out of memory` even when total free bytes look sufficient; mitigations are `torch.cuda.empty_cache()`, the `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` allocator option, gradient checkpointing, and mixed precision (`torch.autocast`).

**`torch.compile` is not free.** First call incurs a compilation cost (seconds to minutes). Frequent recompilation from changing input shapes ("guard" invalidation) or dynamic Python can erase the speedup; excessive **graph breaks** silently drop you back to eager. Diagnose with `TORCH_LOGS="graph_breaks,recompiles"` and prefer static shapes or `dynamic=True` deliberately.

**Non-determinism by default.** Many CUDA kernels (and cuDNN convolutions) are nondeterministic for speed. Reproducibility requires `torch.use_deterministic_algorithms(True)`, fixed seeds across `torch`/`numpy`/Python, and often a performance hit — and some ops have no deterministic implementation and will raise.

**DataLoader multiprocessing.** `num_workers>0` forks worker processes; combined with CUDA-in-workers, non-fork-safe libraries, or large per-sample Python objects, this causes hangs, memory blowups, and shared-memory (`/dev/shm`) exhaustion in containers. Size `--shm-size` accordingly.

**Upgrade friction.** PyTorch does deprecate and remove APIs across minor versions; pickled checkpoints and TorchScript artifacts are not guaranteed portable across versions. Serialize model *state_dict*s (not whole models) and treat the training environment as pinned. `torch.load` on untrusted files is a code-execution risk — recent versions default `weights_only=True`, but older code and `.pt` files from the wild remain a supply-chain hazard.

## When to Use / When Not

**Use when:**
- You are doing ML research or rapid iteration and want models to be plain, debuggable Python.
- You need the largest operator coverage and ecosystem (Transformers, vLLM, Lightning, most pretrained weights target PyTorch).
- You need to scale training across many GPUs (FSDP/DDP) or target NVIDIA/AMD/Apple with one API.
- You want an incremental path from eager prototyping to compiled/exported deployment.

**Avoid when:**
- You need a tiny, dependency-light inference runtime at the edge — a full `torch` install is large; consider ONNX Runtime, ExecuTorch, or a C++/GGML path.
- Your workload is functional-transform-heavy (vmap/grad composition, TPU-first) — JAX is a more natural fit.
- You want maximum-stability, rarely-changing APIs — PyTorch iterates and deprecates across minor versions.
- You are inference-only for LLMs — a serving stack (vLLM, TensorRT-LLM) built *on* PyTorch usually beats hand-rolling.

## Alternatives

- [google/jax](https://github.com/google/jax) — use instead when you want composable function transforms (`grad`/`vmap`/`jit`) and XLA/TPU as first-class; less of a batteries-included NN framework.
- [tensorflow/tensorflow](https://github.com/tensorflow/tensorflow) — use instead when you are invested in the TF/Keras deployment ecosystem (TF Serving, TFLite) or existing TF models.
- [keras-team/keras](https://github.com/keras-team/keras) — use instead for a high-level, multi-backend model API when you don't need low-level control (Keras 3 runs on JAX/TF/PyTorch).
- [ml-explore/mlx](https://github.com/ml-explore/mlx) — use instead for research on Apple Silicon where unified-memory and Metal-native performance matter.
- [tinygrad/tinygrad](https://github.com/tinygrad/tinygrad) — use instead when you want a minimal, hackable framework and are willing to trade ecosystem breadth for simplicity.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017-01 | First public release; define-by-run autograd, Python rewrite of Torch[^1]. |
| 1.0 | 2018-12 | Caffe2 merged in; TorchScript / JIT for production graph capture[^4]. |
| 1.x | 2019–2022 | Stable eager era; distributed (DDP), AMP, mobile, growing backend support. |
| 2.0 | 2023-03 | `torch.compile` (TorchDynamo + AOTAutograd + TorchInductor), largely backward-compatible[^2]. |
| 2.1 | 2023-10 | `torch.compile` maturation, improved dynamic shapes, FSDP improvements. |

## References

[^1]: Adam Paszke et al., "Automatic differentiation in PyTorch" — NeurIPS 2017 Autodiff Workshop; and "PyTorch: An Imperative Style, High-Performance Deep Learning Library," NeurIPS 2019. https://papers.nips.cc/paper/9015-pytorch-an-imperative-style-high-performance-deep-learning-library
[^2]: PyTorch team, "PyTorch 2.0" — 2023. https://pytorch.org/get-started/pytorch-2.0/
[^3]: "PyTorch Foundation" under the Linux Foundation — 2022. https://pytorch.org/foundation
[^4]: PyTorch blog, "The road to 1.0." https://pytorch.org/blog/a-year-in/

## Tags

deep-learning, machine-learning, python, tensor, autograd, gpu, cuda, neural-network, framework, cpp, scientific-computing
