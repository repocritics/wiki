# karpathy/nanoGPT

> The simplest, fastest repository for training/finetuning medium-sized GPTs — a ~600-line, hackable GPT-2 reproduction, now deprecated in favor of nanochat.

[GitHub repo](https://github.com/karpathy/nanoGPT) ·
[License: MIT](https://github.com/karpathy/nanoGPT/blob/master/LICENSE)

## Overview

nanoGPT is Andrej Karpathy's minimal codebase for training and finetuning GPT-style transformers[^1]. It is a rewrite of his earlier teaching repo minGPT that, in his words, "prioritizes teeth over education" — the goal is not the cleanest pedagogy but a real training loop that reproduces GPT-2 (124M) on OpenWebText in about 4 days on a single 8×A100 40GB node. The entire model definition (`model.py`) and training loop (`train.py`) are roughly 300 lines each; that deliberate smallness is the whole point.

The repo occupies a specific niche: it is the reference "from scratch" GPT that people fork to learn, to run ablations, or to bootstrap a research idea without inheriting the weight of a large framework like Megatron or the HuggingFace `Trainer`. It is plain PyTorch, readable top to bottom, and easy to hack. That same minimalism is its ceiling — it has no plugin system, no config abstraction beyond a Python file of overrides, and no ambition to scale past a single node cleanly.

As of November 2025 the maintainer has explicitly marked nanoGPT deprecated in favor of a successor, nanochat, and states it is "now very old" but will be left up "for posterity"[^2]. It remains one of the most-forked ML education repos on GitHub (~10.5k forks), so its patterns are widely copied even though the original is frozen.

## Getting Started

```bash
pip install torch numpy transformers datasets tiktoken wandb tqdm
```

The canonical first run is a character-level GPT on tiny Shakespeare (a 1 MB file):

```bash
# tokenize raw text -> train.bin / val.bin (uint16 token ids)
python data/shakespeare_char/prepare.py

# train a 6-layer / 6-head / 384-dim model, ctx=256 (~3 min on one A100)
python train.py config/train_shakespeare_char.py

# sample from the best checkpoint
python sample.py --out_dir=out-shakespeare-char
```

On a laptop with no GPU, the same config runs with explicit downshifts (`--device=cpu --compile=False --block_size=64 --batch_size=12 --n_layer=4 ...`); on Apple Silicon, `--device=mps` gives a 2–3× speedup over CPU[^1].

## Architecture / How It Works

The repo is four short scripts plus per-dataset `prepare.py` files. There is no package, no installer, no framework layer:

- **`model.py`** — a standard decoder-only transformer: token + learned positional embeddings, a stack of pre-LayerNorm blocks (causal self-attention + MLP with GELU), and a final linear head tied to the embedding. It can instantiate GPT-2 sizes and, critically, `from_pretrained` loads OpenAI's published GPT-2 weights (124M / 350M / 774M / 1558M) by remapping HuggingFace tensors, so you can finetune or evaluate a real GPT-2 rather than only toy models.
- **`train.py`** — a ~300-line loop: gradient accumulation, cosine LR schedule with warmup, AdamW, gradient clipping, mixed precision (`bfloat16`/`float16` autocast + GradScaler), checkpoint save/resume, and optional Weights & Biases logging. Config is a plain Python file whose module-level variables are `exec`'d over the defaults, plus `--key=value` CLI overrides — a deliberately primitive but zero-dependency config system.
- **Distributed training** — via PyTorch DDP through `torchrun`. Single-GPU, single-node-multi-GPU, and multi-node are all the same script with different launch flags. There is no FSDP, tensor parallelism, or pipeline parallelism; the README's own todo list flags FSDP as unimplemented[^1].
- **`sample.py` / `bench.py`** — inference/generation and a stripped benchmarking harness.

Data is pre-tokenized offline into flat `train.bin` / `val.bin` files of raw `uint16` BPE ids (via `tiktoken` for GPT-2 vocab, or a byte/char vocab for the Shakespeare demo). Training reads random windows from these memory-mapped files — there is no data loader abstraction, streaming, or sharding beyond that.

The one non-obvious performance dependency is `torch.compile()`. nanoGPT was written against the then-new PyTorch 2.0 (Dec 2022) and relies on `compile` to hit its quoted throughput — the README notes it cut iteration time from ~250 ms to ~135 ms[^1].

## Production Notes

nanoGPT is a research/education base, not production infrastructure, and should be read that way. Specific caveats:

- **Deprecated upstream.** No new features will land. Treat the repo as a frozen reference; for an actively maintained sibling, the author points to nanochat[^2].
- **`torch.compile` is a footgun on some platforms.** It is on by default and was not supported on Windows at release; the documented fix is `--compile=False`, which runs but forfeits a large fraction of the speedup[^1].
- **Multi-node without InfiniBand crawls.** The README explicitly warns to benchmark interconnect and to set `NCCL_IB_DISABLE=1` when IB is absent — training will run but "most likely _crawl_"[^1].
- **The dataset domain gap is real.** Reproducing GPT-2 uses OpenWebText, an open approximation of OpenAI's closed WebText. A raw GPT-2 124M checkpoint scores ~3.11 val loss on OWT; finetuning closes it to ~2.85. Compare like-with-like or the "reproduction" looks worse than it is[^1].
- **No config validation, no tests to speak of.** Because config is `exec`'d Python, a typo in an override silently does nothing or crashes mid-run. There is no CI gate protecting forks.
- **Checkpoint format is naive.** Optimizer buffers are stored alongside model params in one blob (the author's own todo notes wanting to separate them), which makes checkpoints larger and less portable than necessary.
- **Scope is genuinely "medium-sized GPTs."** It stops where serious scale begins — no FSDP/ZeRO, no rotary/ALiBi embeddings (learned absolute only), no activation checkpointing beyond what PyTorch gives you.

## When to Use / When Not

**Use when:**
- You want to learn how GPT pretraining actually works, end to end, from readable code.
- You need a small, hackable base for research ablations or a novel architecture experiment on ≤1 node.
- You want to finetune or evaluate real GPT-2 checkpoints with minimal ceremony.

**Avoid when:**
- You need a maintained, supported training stack — it is officially deprecated.
- You are training models beyond a single node's worth of GPUs, or need FSDP/tensor-parallel scaling.
- You want turnkey modern features (rotary embeddings, FlashAttention config, LoRA, quantized finetuning) without writing them yourself.
- You want a production inference server — this is a `sample.py`, not a serving system.

## Alternatives

- karpathy/nanochat — the author's own successor; use it instead for anything new.
- karpathy/minGPT — the education-first predecessor; use it when clarity matters more than training throughput.
- karpathy/llm.c — GPT-2/GPT-3 training in raw C/CUDA; use when you want to understand the kernels, not the Python.
- huggingface/transformers — use when you need breadth, maintained models, and an ecosystem rather than a single hackable file.
- pytorch/torchtitan — use when you need production-grade multi-node scaling (FSDP2, tensor/pipeline parallelism) for large models.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-12-28 | Repo created; rewrite of minGPT targeting GPT-2 reproduction[^1]. |
| — | 2022-12-29 | Written against PyTorch 2.0 `torch.compile` (nightly at the time)[^1]. |
| — | 2025-11 | Marked deprecated in favor of nanochat; left up "for posterity"[^2]. |
| — | 2025-11-12 | Last recorded push to `master`. |

nanoGPT is unversioned — there are no tagged releases; it is consumed as a repo to fork rather than a package to depend on.

## References

[^1]: nanoGPT README, karpathy/nanoGPT. https://github.com/karpathy/nanoGPT/blob/master/README.md
[^2]: nanoGPT README, "Update Nov 2025" deprecation notice pointing to nanochat. https://github.com/karpathy/nanochat

## Tags

python, pytorch, gpt, transformer, language-model, machine-learning, deep-learning, training, education, deprecated
