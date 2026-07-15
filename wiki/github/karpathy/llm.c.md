# karpathy/llm.c

> GPT-2/GPT-3 training implemented directly in C/CUDA, with no PyTorch and no Python in the hot path — a readable reference for how LLM pretraining actually works at the metal.

[GitHub repo](https://github.com/karpathy/llm.c) ·
No official website ·
[License: MIT](https://github.com/karpathy/llm.c/blob/master/LICENSE)

## Overview

llm.c is Andrej Karpathy's implementation of GPT-2/GPT-3-style transformer pretraining written in plain C and CUDA, with no dependency on PyTorch (~245MB) or the CPython runtime (~107MB)[^1]. It began in April 2024 as the CUDA-and-C successor to his earlier PyTorch project nanoGPT[^2] — the `train_gpt2.py` reference file in the repo is a lightly tweaked nanoGPT kept in sync with the C code. The goal is twofold and in tension: be a single-file, readable teaching artifact, and still be fast enough to do real pretraining runs. At release the mainline CUDA path was reported to run roughly 7% faster than PyTorch nightly on the same GPT-2 workload[^1].

The project's defining tradeoff is manual everything. There is no autograd: every layer (encoder, LayerNorm, matmul, attention, GELU, residual, softmax, cross-entropy) has a hand-written forward and backward pass. Parameters and activations live in single flat buffers indexed by pointer arithmetic rather than tensor objects. This makes the entire training loop legible end-to-end — you can read the whole thing — but it also means the architecture is effectively hardcoded to the GPT-2/GPT-3 dense transformer. Changing the model means editing C, not a config.

llm.c is best read as an educational reference and a cheap-reproduction tool, not a general framework. As of mid-2026 the repo is popular (30k+ stars) but effectively dormant: the last substantive commits landed in mid-2025 and there are no tagged releases[^3]. Treat it as a frozen, well-understood artifact to learn from rather than a maintained dependency.

## Getting Started

There is no package manager — you build with `make` and `nvcc`/`gcc`. The CPU reference path (toy speed, but no GPU required):

```bash
git clone https://github.com/karpathy/llm.c
cd llm.c
chmod u+x ./dev/download_starter_pack.sh
./dev/download_starter_pack.sh      # GPT-2 124M weights, tokenizer, tinyshakespeare .bin
make train_gpt2
OMP_NUM_THREADS=8 ./train_gpt2       # finetunes GPT-2 124M on tinyshakespeare, CPU
```

The GPU mainline (mixed precision bf16, requires an NVIDIA GPU + CUDA toolkit):

```bash
make train_gpt2cu                    # add USE_CUDNN=1 to enable cuDNN flash attention
./train_gpt2cu                       # single GPU
# multi-GPU on one node (needs NCCL + MPI):
mpirun -np 8 ./train_gpt2cu
```

Datasets are pre-tokenized into a custom `.bin` format (1024-byte header + a stream of uint16 GPT-2 token ids) by Python scripts under `dev/data/`, e.g. `python dev/data/tinyshakespeare.py`. The Python is a build-time/data-prep convenience, not part of the training loop.

## Architecture / How It Works

The repo is deliberately organized around a few self-contained files rather than a module tree:

- **`train_gpt2.c`** — the CPU, fp32 reference in roughly 1,000 lines. This is the canonical "read this to understand the whole thing" file.
- **`train_gpt2.cu`** — the mainline CUDA implementation: mixed precision (bf16 with fp32 master weights), gradient accumulation, AdamW, multi-GPU. This is where performance work lands.
- **`train_gpt2fp32.cu`** — a frozen early CUDA checkpoint kept as a simpler, more portable fp32-only path for people learning CUDA.
- **`dev/cuda/`** — a library of the same kernels written at escalating levels of optimization (naive → cuBLAS-competitive), explicitly for education. Each kernel is benchmarked so you can say "my hand-written matmul is 80% of cuBLAS."

The training step is the classic loop made explicit: forward pass writes every intermediate activation into one large pre-allocated buffer; the backward pass walks the same layers in reverse writing gradients into a parallel buffer; AdamW updates the parameter buffer in place. Because activation memory is a single allocation sized up front, there is no dynamic graph and no allocator churn during training.

For speed the mainline pulls in the standard NVIDIA stack as single, interpretable dependencies: **cuBLAS/cuBLASLt** for matmuls and **cuDNN** for Flash Attention (added May 1, 2024, off by default because it inflates compile time from seconds to ~a minute)[^4]. Multi-GPU uses **NCCL** for gradient all-reduce; multi-node initialization is offered three ways — OpenMPI, a shared filesystem, or TCP sockets — because NCCL bootstrap depends on the cluster environment.

Karpathy's stated curation rule is the architecture: the root folder stays simple and readable, and a PR that adds 2% speed for 500 lines of complex C or an exotic dependency gets rejected; that complexity is pushed into `dev/` instead[^1]. This is why the mainline stays legible while still being fast.

## Production Notes

llm.c is not a production training framework, and using it as one is the main footgun. Real caveats:

- **Dormant / unmaintained.** Last meaningful commits were mid-2025; ~220 issues sit open and there are no releases[^3]. Do not build a pipeline on it expecting upstream fixes. Fork and freeze if you depend on it.
- **NVIDIA-only fast path.** The performant route is CUDA + bf16. The CPU path exists for demonstration and is far too slow for real training. AMD/Metal/OpenCL support exists only in third-party forks, not mainline[^1].
- **Hardcoded to GPT-2/GPT-3 dense transformers.** No config surface for other architectures. Adding a layer means writing both its forward and its backward kernel by hand — high friction for research modification, which is the point of a teaching repo but a wall for a lab.
- **Build friction.** You must match the CUDA toolkit version, and cuDNN Flash Attention additionally requires the cuDNN library plus the header-only `cudnn-frontend` on a path the Makefile can find (`CUDNN_FRONTEND_PATH=...`). Enabling it roughly turns a few-second build into a ~minute build.
- **Multi-node bootstrap is environment-sensitive.** The README specifically warns that Slurm builds without PMIx support force you onto the filesystem or TCP init paths; `srun --mpi=list` to check[^1].
- **Reproduction economics.** The headline result is reproducing GPT-2 (124M) — documented step-by-step in Discussion #481 — on a single 8×A100 node in roughly 90 minutes for about $20 of cloud time[^5]. Larger GPT-2/GPT-3 models were also demonstrated; cost and wall-clock scale with model size accordingly.

## When to Use / When Not

**Use when:**
- You want to understand transformer pretraining end-to-end without framework abstractions in the way.
- You're learning CUDA kernel authoring and want a benchmarked, well-documented kernel ladder (`dev/cuda/`).
- You want to cheaply reproduce GPT-2 from scratch as a teaching or baseline exercise.
- You want a minimal, dependency-light C/CUDA training loop to fork and specialize.

**Avoid when:**
- You need to train novel or non-GPT architectures, or iterate on model design quickly — hand-written backward passes make this slow.
- You're on non-NVIDIA hardware and want first-class support.
- You need an actively maintained, supported training stack for production.
- You want autograd, distributed abstractions, checkpointing ergonomics, and dataloaders out of the box.

## Alternatives

- karpathy/nanoGPT — the PyTorch predecessor; use it when you want the same GPT-2 training in readable Python you can modify without writing CUDA.
- pytorch/pytorch — use when you need autograd, arbitrary architectures, and a maintained ecosystem rather than a fixed reference.
- NVIDIA/Megatron-LM — use for production-scale (billions of parameters, many nodes) transformer training with tensor/pipeline parallelism.
- tinygrad/tinygrad — use when you want a small, hackable framework that still targets multiple backends (not just NVIDIA).
- ggml-org/llama.cpp — use when your goal is running/inference of trained models in C/C++ on commodity hardware, not training.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Initial release | 2024-04-08 | CPU fp32 reference in ~1,000 lines of C, successor to nanoGPT[^2]. |
| CUDA mainline | 2024-04 | `train_gpt2.cu`, mixed precision, cuBLAS matmuls, multi-GPU via NCCL. |
| cuDNN Flash Attention | 2024-05-01 | Optional (`USE_CUDNN=1`), off by default due to compile cost[^4]. |
| GPT-2 124M reproduction | 2024-05 | Documented in Discussion #481: ~90 min, ~$20 on 8×A100[^5]. |
| GPT-2/GPT-3 miniseries | 2024 | Larger models reproduced; parallel PyTorch reference kept in sync. |
| Effectively dormant | 2025-06 | Last substantive commits; no tagged releases since[^3]. |

## References

[^1]: llm.c README, karpathy/llm.c. https://github.com/karpathy/llm.c/blob/master/README.md
[^2]: karpathy/nanoGPT — the PyTorch predecessor project. https://github.com/karpathy/nanoGPT
[^3]: GitHub API metadata for karpathy/llm.c (created 2024-04-08, last push 2025-06-26, no releases, MIT), retrieved 2026-07-15. https://github.com/karpathy/llm.c
[^4]: "flash attention" section of the README (cuDNN Flash Attention added 2024-05-01). https://github.com/karpathy/llm.c/blob/master/README.md
[^5]: Discussion #481, "Reproducing GPT-2 (124M) in llm.c". https://github.com/karpathy/llm.c/discussions/481

## Tags

c, cuda, machine-learning, llm, gpt-2, transformers, deep-learning, training, gpu, education, nvidia
