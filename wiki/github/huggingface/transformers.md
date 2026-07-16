# huggingface/transformers

> The model-definition framework for state-of-the-art ML across text, vision, audio, and multimodal — the shared source of truth that training and inference engines build on.

[GitHub repo](https://github.com/huggingface/transformers) ·
[Official website](https://huggingface.co/transformers) ·
[License: Apache-2.0](https://github.com/huggingface/transformers/blob/main/LICENSE)

## Overview

Transformers is a Python library of reference implementations for machine-learning model architectures, plus the loading, tokenization, and generation code to run them. It began in late 2018 as `pytorch-pretrained-bert` — a single-model port of Google's BERT — was renamed `pytorch-transformers` in 2019, then simply `transformers` with the 2.0 release that added TensorFlow 2 support[^1]. It is one of the most-starred repositories on GitHub (over 162k stars, ~34k forks) and remains actively maintained, with commits landing daily.

The library's strategic role has shifted. It is no longer positioned mainly as "a way to run models," but as the canonical *model definition* layer for the wider ecosystem: if an architecture's forward pass is defined in `transformers`, downstream training frameworks (Axolotl, Unsloth, DeepSpeed, FSDP) and inference engines (vLLM, SGLang, TGI, llama.cpp, MLX) can reuse that definition rather than re-implementing it[^2]. This makes the repo an ecosystem chokepoint — new model support here unblocks the rest of the stack — and explains why vendors race to land their architectures in it on launch day.

The defining tension is deliberate anti-abstraction. Each model lives in its own self-contained file with the forward pass spelled out in full, even when that means copying attention or normalization code across dozens of models. This "repeat yourself" stance[^3] optimizes for researchers reading and forking one model in isolation, at the cost of DRY engineering: a fix in one model does not automatically propagate to its copies.

## Getting Started

```bash
pip install "transformers[torch]"
# or, with the uv package manager:
uv pip install "transformers[torch]"
```

Transformers requires Python 3.10+ and PyTorch 2.4+. The high-level `pipeline` API handles preprocessing, model loading, and decoding:

```python
from transformers import pipeline

pipe = pipeline(task="text-generation", model="Qwen/Qwen2.5-1.5B")
print(pipe("The secret to baking a good cake is"))
```

For chat models, pass a message list; the tokenizer applies the model's chat template:

```python
import torch
from transformers import pipeline

chat = [
    {"role": "system", "content": "You are a terse assistant."},
    {"role": "user", "content": "Name three things to do in New York."},
]
pipe = pipeline(
    task="text-generation",
    model="meta-llama/Meta-Llama-3-8B-Instruct",
    dtype=torch.bfloat16,
    device_map="auto",
)
print(pipe(chat, max_new_tokens=256)[0]["generated_text"][-1]["content"])
```

Under the pipeline, the core is three classes — `AutoConfig`, `AutoTokenizer`, `AutoModel*` — each with a `from_pretrained()` that resolves and caches weights from the Hugging Face Hub.

## Architecture / How It Works

The library is organized around three composable primitives:

1. **Configuration** (`PretrainedConfig`) — hyperparameters that fully specify an architecture (hidden size, layer count, vocab).
2. **Tokenizer** — text↔token conversion. "Fast" tokenizers are backed by the Rust `tokenizers` library; "slow" ones are pure Python. Behavior can differ subtly between the two.
3. **Model** (`PreTrainedModel`, a `torch.nn.Module`) — the network itself, plus task heads (`*ForCausalLM`, `*ForSequenceClassification`, etc.).

The `Auto*` classes dispatch to the right concrete class by reading `config.json`'s `model_type`. `from_pretrained()` downloads weights (defaulting to `safetensors` when available), maps them onto the module, and places tensors according to `device_map` and `dtype`.

**Weight sharding and offload** are delegated to `accelerate`. `device_map="auto"` splits a model across GPUs, CPU, and disk when it exceeds device memory — the mechanism that makes multi-billion-parameter models loadable on modest hardware, at the cost of transfer overhead.

**Attention is pluggable** via `attn_implementation`: `eager` (pure PyTorch, most portable), `sdpa` (PyTorch's fused scaled-dot-product attention, the default when available), and `flash_attention_2` / `flash_attention_3` (external kernels for long-context throughput, requiring a separate install and compatible GPU).

**Generation** lives in a `generate()` mixin implementing greedy, beam, sampling, and assisted/speculative decoding, plus a KV cache. It is correct and flexible but single-sequence-oriented; it is not a batched, paged-attention serving stack.

Historically the same architectures were mirrored across PyTorch (`ModelName`), TensorFlow (`TFModelName`), and Flax/JAX (`FlaxModelName`). Maintenance has since consolidated toward PyTorch-first, with TensorFlow and Flax support de-emphasized[^2]. New models generally ship PyTorch-only.

## Production Notes

**It is not a serving engine.** `model.generate()` is convenient but leaves throughput on the table versus vLLM, SGLang, or TGI, which add continuous batching and paged KV cache. Use Transformers to *define and validate* a model; use a dedicated engine to serve it at scale. The 2025-era repositioning as a "model-definition framework" is partly an acknowledgment of this division of labor[^2].

**Version churn is real.** Minor releases can change defaults and deprecate arguments (for example, `torch_dtype` → `dtype`). Pin an exact version in production and read release notes before upgrading; code written against one minor version can warn or break on the next.

**Copied model code cuts both ways.** The self-contained-file design means a bug or optimization fixed in one architecture is not automatically inherited by architectures that copied it. The repo mitigates this with `# Copied from` markers and a consistency check, but the guarantee is tooling-enforced, not structural.

**`trust_remote_code` is an execution risk.** Some Hub models ship custom modeling code that only runs if you pass `trust_remote_code=True`, which executes arbitrary downloaded Python. Treat it like running an untrusted script — pin a specific revision and review the code.

**Cache and disk.** `from_pretrained()` caches to `HF_HOME` (default `~/.cache/huggingface`). Large-model workflows fill disks quickly; set `HF_HOME` deliberately in containers and CI, and pre-download weights rather than fetching at request time.

**Tokenizer parity.** Fast and slow tokenizers can produce different outputs on edge cases (special tokens, whitespace). Fix `use_fast` explicitly when reproducibility across environments matters.

**Memory footprint.** Load in `bfloat16`/`float16` where supported, and reach for `bitsandbytes` 4/8-bit quantization for inference on constrained GPUs; loading in the default `float32` doubles memory versus what most modern checkpoints actually need.

## When to Use / When Not

**Use when:**
- You need to load, fine-tune, or experiment with a specific pretrained architecture from the Hub.
- You want a single, uniform API across text, vision, audio, and multimodal models.
- You are prototyping, evaluating, or writing training loops (via `Trainer` or `accelerate`).
- You need the canonical reference implementation of an architecture to port or study.

**Avoid when:**
- You are building a high-throughput inference service — reach for vLLM, SGLang, or TGI instead.
- You want a small, dependency-light runtime for a single quantized model on CPU — llama.cpp is leaner.
- You need a modular neural-net toolbox of reusable building blocks; the anti-abstraction design is the opposite of that.
- You require a stable, rarely-changing API surface; the library iterates fast and deprecates regularly.

## Alternatives

- vllm-project/vllm — use instead when the goal is serving throughput and latency, not model definition; consumes Transformers-defined models.
- ggml-org/llama.cpp — use instead for local, quantized CPU/GPU inference with minimal dependencies.
- huggingface/diffusers — use instead for diffusion and image/video generation models, which Transformers does not cover.
- unslothai/unsloth — use instead when you want faster, lower-memory LoRA/QLoRA fine-tuning of the same models.
- pytorch/pytorch — use directly when you need full control and are writing an architecture from scratch rather than reusing one.

## History

| Version | Date | Notes |
|---------|------|-------|
| pytorch-pretrained-bert | 2018-11 | Initial release: a PyTorch port of BERT[^1]. |
| pytorch-transformers 1.0 | 2019-07 | Renamed; multiple architectures (GPT, GPT-2, XLNet, XLM)[^1]. |
| 2.0 | 2019-09 | Renamed to `transformers`; TensorFlow 2 support added[^1]. |
| 3.0 | 2020-06 | Fast (Rust) tokenizers, pipelines maturation. |
| 4.0 | 2020-11 | Python 3 only; API cleanup; EMNLP 2020 paper published[^4]. |
| (ongoing) | 2024–2026 | PyTorch-first consolidation; repositioned as the ecosystem's model-definition layer[^2]. |

## References

[^1]: Hugging Face, "transformers" release history and naming — `pytorch-pretrained-bert` → `pytorch-transformers` → `transformers`. https://github.com/huggingface/transformers/releases
[^2]: Transformers README, "the model-definition framework" and cross-ecosystem pivot (Axolotl, vLLM, SGLang, TGI, llama.cpp, MLX). https://github.com/huggingface/transformers/blob/main/README.md
[^3]: Hugging Face blog, "Don't Repeat Yourself" / the single-model-file philosophy. https://huggingface.co/blog/transformers-design-philosophy
[^4]: Wolf et al., "Transformers: State-of-the-Art Natural Language Processing," EMNLP 2020 (System Demonstrations). https://aclanthology.org/2020.emnlp-demos.6/

## Tags

python, pytorch, machine-learning, deep-learning, nlp, llm, transformers, pretrained-models, multimodal, model-hub, inference, training
