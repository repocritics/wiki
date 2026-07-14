# unslothai/unsloth

> Triton-kernel-accelerated fine-tuning for open LLMs — 2x faster, lower VRAM, single-GPU-first — now wrapped in a local web UI (Unsloth Studio).

[GitHub repo](https://github.com/unslothai/unsloth) ·
[Official website](https://unsloth.ai/docs) ·
[License: Apache-2.0](https://github.com/unslothai/unsloth/blob/main/LICENSE)

## Overview

Unsloth is a fine-tuning acceleration library for open large language models, built by Daniel and Michael Han and first released in late 2023[^1]. Its core value is a set of hand-written Triton kernels and a manual autograd path that replace the hot operations inside Hugging Face `transformers` (attention, MLP, RoPE, RMSNorm, cross-entropy) during LoRA / QLoRA training. The headline claim — roughly 2x faster training with up to ~70% less VRAM and no accuracy loss — applies to single-GPU LoRA/QLoRA workloads, which is where almost all of the engineering is concentrated[^2].

The defining tension is **coverage vs. speed via monkey-patching**. Unsloth does not reimplement a training framework; it patches specific model architectures in `transformers` and `peft` at import time. That means each supported model family (Llama, Mistral, Gemma, Qwen, Phi, DeepSeek, gpt-oss, and vision/TTS/embedding variants) needs deliberate per-architecture support, and Unsloth is tightly pinned to the `transformers`/`peft`/`bitsandbytes` versions it was built against. When a new base model drops, Unsloth is often first to ship a working (and bug-fixed) path — the project is well known for catching tokenizer and chat-template bugs in upstream model releases — but a model or a `transformers` version it hasn't patched simply won't get the speedups, or may fail to load.

As of 2026 the repository has grown a second front end, **Unsloth Studio** (Beta): a local web UI that bundles model search/download, chat inference (GGUF, LoRA, safetensors), export, and a visual training/data workflow on top of the same core[^3]. The underlying **Unsloth Core** Python library remains the code-based path most training pipelines actually import. Historically the open-source library was single-GPU; the README now lists multi-GPU as available with further work in progress[^3].

At time of writing the repo has ~68.2k stars, ~6.1k forks, and 953 open issues, under Apache-2.0[^4].

## Getting Started

Unsloth Core (the code-based library) via `uv`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv unsloth_env --python 3.13
source unsloth_env/bin/activate
uv pip install unsloth --torch-backend=auto
```

Minimal 4-bit QLoRA setup with the `FastLanguageModel` API:

```python
from unsloth import FastLanguageModel   # import BEFORE transformers/trl

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name     = "unsloth/llama-3.1-8b-bnb-4bit",
    max_seq_length = 2048,
    load_in_4bit   = True,
)

model = FastLanguageModel.get_peft_model(
    model,
    r              = 16,
    lora_alpha     = 16,
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                      "gate_proj", "up_proj", "down_proj"],
    use_gradient_checkpointing = "unsloth",   # Unsloth's long-context checkpointing
)
# `model` is a standard PEFT model — hand it to trl's SFTTrainer / GRPOTrainer.
```

Or the Studio web UI (installs a self-contained app):

```bash
curl -fsSL https://unsloth.ai/install.sh | sh   # macOS / Linux / WSL
unsloth studio -p 8888
```

## Architecture / How It Works

Unsloth is a **patch layer**, not a framework. Importing `unsloth` (before `transformers` and `trl`) swaps in optimized implementations:

- **Custom Triton kernels** for the attention/MLP/RoPE/RMSNorm/cross-entropy paths, with a manually written backward pass rather than relying on autograd over the reference PyTorch ops. This is where the speed and VRAM wins come from — fused kernels avoid materializing intermediate activations.
- **Architecture-specific model classes** (`FastLanguageModel`, `FastVisionModel`, etc.) that recognize a known model config and route it through the patched path. Unknown architectures fall back or error.
- **4-bit quantization via bitsandbytes** for QLoRA; Unsloth also publishes its own "dynamic" GGUF and 4-bit quants on Hugging Face that selectively keep sensitive layers at higher precision.
- **Gradient checkpointing rewrite** (`use_gradient_checkpointing="unsloth"`) that offloads/recomputes to push context length far past the stock implementation on the same card.

Because the output of `get_peft_model` is an ordinary PEFT model, Unsloth composes with `trl`'s `SFTTrainer`, `DPOTrainer`, and `GRPOTrainer` — RL (GRPO/GSPO, FP8) is a first-class use case, not a bolt-on. Export paths cover merged 16-bit safetensors and GGUF (it drives a local `llama.cpp` build for the latter).

The coupling story is the whole story: Unsloth lives or dies by staying in lockstep with `transformers`, `peft`, `bitsandbytes`, `triton`, and CUDA. Version drift between these is the most common source of breakage. Studio isolates this somewhat by shipping a pinned, self-contained environment (its own venv, `studio.db`, and `llama.cpp` build under `~/.unsloth/studio`).

## Production Notes

- **"Import unsloth first" is load-bearing.** The patches must apply before `transformers`/`trl` are imported. Import-order bugs manifest as the fast path silently not activating (you get correct results at stock speed) — easy to miss.
- **Single-GPU is the well-trodden path.** Multi-GPU is now listed as supported, but the open-source project's depth, docs, and community reports are overwhelmingly single-GPU. For serious multi-node/FSDP/DeepSpeed runs, frameworks built for distribution (Axolotl, LLaMA-Factory) are more battle-tested. Treat Unsloth multi-GPU as maturing.
- **Version pinning is not optional.** Upgrading `transformers` or `bitsandbytes` out from under a working Unsloth install is a frequent breakage. Pin the trio together; prefer Unsloth's own install command and pre-quantized `unsloth/*-bnb-4bit` model repos.
- **Speed/VRAM numbers are workload-specific.** The 2x/70% figures are LoRA/QLoRA on supported models at particular sequence lengths. Full fine-tuning, unsupported architectures, or very short sequences see smaller gains. Benchmark your own config rather than trusting the marketing table.
- **New-model lead time cuts both ways.** Unsloth often ships day-one support with upstream tokenizer/chat-template bug fixes — a real reason to use it early. But that same speed means occasional regressions; nightly (`main`) is where fixes land first and where breakage does too.
- **Studio exposes code execution.** Studio's server-side tools (web search, Python/terminal execution) run as your user and are on by default. Binding with `-H 0.0.0.0` or `--secure` publishes it (the latter via a Cloudflare tunnel); anyone with the URL and API key can run code on the host. Keep the API key private and use `--disable-tools` when exposing it.
- **Accuracy-loss claims deserve your own eval.** "No accuracy degradation" is plausible for the LoRA math but should be verified on your metric — custom kernels and quantization interact with numerics in ways that are model- and task-dependent.

## When to Use / When Not

**Use when:**
- You're doing LoRA/QLoRA fine-tuning or GRPO-style RL on a single consumer or datacenter GPU and want the best VRAM/speed envelope.
- You want day-one support for a freshly released open model, often with upstream bugs already fixed.
- You want a low-friction local path (Colab/Kaggle notebooks, or the Studio UI) to run and train without assembling a stack.
- You need to fit longer context or a bigger model onto one card than stock `transformers` allows.

**Avoid when:**
- Your model architecture isn't on Unsloth's supported list — you'll get no speedup or a hard failure.
- You need mature, first-class multi-node / FSDP / DeepSpeed distributed training today.
- You want a dependency-light, framework-native pipeline — Unsloth's monkey-patching and tight version pins are a maintenance liability in locked-down environments.
- You're only serving/quantizing models, not training them — go straight to llama.cpp / vLLM.

## Alternatives

- huggingface/peft — the reference LoRA/QLoRA library; use it when you want maximum model coverage and stock reliability over Unsloth's kernel speedups.
- hiyouga/LLaMA-Factory — broad training UI/CLI across many models and methods with first-class multi-GPU; use when breadth and distributed runs matter more than single-GPU throughput.
- axolotl-ai-cloud/axolotl — YAML-configured fine-tuning with strong FSDP/DeepSpeed support; use for production multi-node training.
- pytorch/torchtune — native-PyTorch, dependency-light fine-tuning; use when you want to avoid the `transformers` patch stack.
- ggml-org/llama.cpp — GGUF inference and quantization; use when you only need to run or quantize models, not train them.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-11 | Repo created; first public "2x faster LoRA" release via Triton kernels[^1][^4]. |
| — | 2024 | Broad model support (Llama 3, Mistral, Gemma, Phi, Qwen); dynamic 4-bit/GGUF quants; day-one bug fixes for major model launches[^2]. |
| — | 2025 | GRPO/RL path, FP8 and vision RL, long-context (500K) training work, faster MoE training[^3]. |
| Studio (Beta) | 2026 | Unsloth Studio local web UI (run/train/export); multi-GPU listed as available; API inference endpoint[^3]. |

## References

[^1]: Unsloth, "Introducing Unsloth" / initial release announcement (late 2023). https://unsloth.ai/blog
[^2]: Unsloth documentation — fine-tuning guide and benchmarks. https://unsloth.ai/docs/get-started/fine-tuning-llms-guide
[^3]: unslothai/unsloth README (Unsloth Studio + Core, features and news), retrieved 2026-07. https://github.com/unslothai/unsloth
[^4]: GitHub API — repos/unslothai/unsloth metadata (stars, forks, license, created/pushed dates), retrieved 2026-07-15. https://api.github.com/repos/unslothai/unsloth

## Tags

python, llm, fine-tuning, lora, qlora, triton-kernels, reinforcement-learning, gguf, machine-learning, self-hosted
