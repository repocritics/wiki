# huggingface/peft

> Parameter-Efficient Fine-Tuning for PyTorch — train a few million adapter weights instead of a full model, and ship checkpoints measured in megabytes.

[GitHub repo](https://github.com/huggingface/peft) ·
[Official website](https://huggingface.co/docs/peft) ·
[License: Apache-2.0](https://github.com/huggingface/peft/blob/main/LICENSE)

## Overview

PEFT is a Hugging Face library that adapts large pretrained models by training a small set of extra (or reparameterized) parameters while the original weights stay frozen. The canonical method is LoRA[^1] — inject a pair of low-rank matrices into selected linear layers, train only those, and either keep them as a detachable adapter or merge them back into the base weights. The result: a fine-tune that fits on consumer GPUs and produces checkpoints of a few megabytes instead of the multi-gigabyte artifact a full fine-tune generates.

The library is less a framework than a collection of adapter implementations behind a uniform API. `get_peft_model(base_model, config)` walks a `torch.nn.Module`, finds the modules named in the config, and swaps them for adapter-wrapped versions. Everything else — the training loop, the tokenizer, the optimizer — is still plain PyTorch or Transformers. PEFT deliberately owns only the adapter layer, which is why it composes with Transformers, Diffusers, Accelerate, and TRL rather than replacing them[^2].

The defining tension is coverage versus stability. PEFT tracks the research frontier closely — LoRA, QLoRA, DoRA, IA3, VeRA, LoHa/LoKr, OFT/BOFT, prefix/prompt/P-tuning, and more arrive as new methods are published[^3] — but that breadth means many code paths are exercised mostly by a single method on a single model family, and correct results depend heavily on choosing the right `target_modules` for your architecture.

## Getting Started

```bash
pip install peft
```

```python
from transformers import AutoModelForCausalLM
from peft import LoraConfig, TaskType, get_peft_model

model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-3B-Instruct", device_map="cuda")

config = LoraConfig(
    r=16,                 # rank of the low-rank update
    lora_alpha=32,        # scaling; effective scale is alpha / r
    lora_dropout=0.05,
    task_type=TaskType.CAUSAL_LM,
    target_modules=["q_proj", "v_proj"],  # which linears to adapt
)

model = get_peft_model(model, config)
model.print_trainable_parameters()
# trainable params: ~3.7M || all params: ~3.09B || trainable%: ~0.12

# ... train with transformers Trainer / TRL / a custom loop ...
model.save_pretrained("qwen-lora")   # writes only the adapter (a few MB)
```

Loading for inference re-attaches the adapter to the original base model:

```python
from peft import PeftModel
base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-3B-Instruct", device_map="cuda")
model = PeftModel.from_pretrained(base, "qwen-lora")
```

## Architecture / How It Works

PEFT's core operation is **module replacement**. `get_peft_model` returns a `PeftModel` wrapper; inside it, a tuner (e.g. `LoraModel`) traverses the base model and, for every submodule whose name matches `target_modules`, replaces it with an adapter layer that holds a reference to the frozen original plus the trainable additions. For LoRA on an `nn.Linear`, the forward pass becomes `W x + (alpha/r) * B (A x)`, where `A` and `B` are the low-rank matrices and `W` is frozen[^1]. The base weights are never copied — only wrapped — so memory overhead during training is dominated by optimizer state on the small adapter, not the model.

`target_modules` is the load-bearing config field. It can be an explicit list of leaf module names, a regex, or the string `"all-linear"`. PEFT ships default target sets for common architectures, but for a model it doesn't recognize you must supply targets yourself; a wrong or empty match trains nothing and fails silently in terms of loss behavior.

Adapters are **detachable and mergeable**. `save_pretrained` serializes only the adapter tensors (plus an `adapter_config.json`), so a checkpoint is independent of and much smaller than the base. At inference you can either keep the adapter live (a small per-layer compute cost) or call `merge_and_unload()` to fold `(alpha/r) * B A` into `W` and recover a plain model with zero adapter overhead. A single `PeftModel` can hold **multiple named adapters** and switch between them with `set_adapter`, which is what enables serving many task-specific LoRAs against one shared base.

**Quantization** is a first-class combination, not an afterthought. QLoRA[^4] loads the base model in 4-bit (NF4) via bitsandbytes, keeps it frozen and quantized, and trains LoRA adapters in higher precision on top. `prepare_model_for_kbit_training` sets up the frozen quantized model (casting layernorms, enabling gradient checkpointing, making inputs require grad) before adapters are attached. This is the path that puts 7B–13B fine-tuning on a single 16–24GB GPU.

Beyond LoRA, the same wrapper machinery hosts structurally different methods: **IA3** (learned per-activation rescaling vectors), **prefix/prompt/P-tuning** (trainable virtual tokens prepended in the attention/embedding space rather than weight deltas), and reparameterizations like **DoRA**, **VeRA**, **LoHa**, **LoKr**, and **OFT/BOFT**. They share the config/save/load surface but have very different memory profiles and mergeability guarantees.

## Production Notes

**`target_modules` is the number-one footgun.** On an unrecognized architecture, an empty or mismatched target set produces a model that trains but learns nothing useful. Verify with `print_trainable_parameters()` — if the trainable count is implausibly small or the percentage is zero, your targets didn't match. `"all-linear"` is a safer default than guessing names, at the cost of more parameters.

**Merging is not always lossless or even available.** `merge_and_unload` on a full-precision base is a clean addition, but merging LoRA into a **4-bit/8-bit quantized** base is lossy or unsupported depending on the quant backend, because you cannot exactly represent `W + BA` in the quantized format. For quantized bases, the common pattern is: merge into a dequantized copy, or serve the adapter unmerged. Expect small metric shifts after merging even in full precision.

**Adapters are useless without the exact base model.** `save_pretrained` stores only the delta plus a pointer to the base model ID in `adapter_config.json`. If that base is a gated/private repo, moves, or changes revision, your adapter no longer loads reproducibly. For durable artifacts, pin the base model revision and consider shipping a merged checkpoint for anything that must survive independently.

**bitsandbytes ties QLoRA to CUDA.** The 4-bit path depends on bitsandbytes, whose support outside NVIDIA GPUs (CPU, Apple Silicon, ROCm) has historically lagged or required specific builds. QLoRA on non-CUDA hardware is not a smooth default path; plan for CPU offloading, a different quantizer, or full-precision LoRA instead.

**Rank and scaling matter more than people expect.** The effective update scale is `lora_alpha / r`; changing `r` without adjusting `alpha` silently changes the learning dynamics. Too-low rank underfits; too-high rank erodes the parameter savings that are the whole point. There is no universal setting — it is per-model, per-task tuning.

**API and config churn.** PEFT moves with the research frontier, and adapter config fields and method-specific options evolve across releases. Adapters saved with an old version can require care to load in a newer one; keep the PEFT version alongside the base model revision in your reproducibility notes.

**Serving many adapters.** Hot-swapping named adapters against one base is memory-efficient but the switch is not free, and true concurrent multi-adapter batching (different requests hitting different LoRAs in one batch) is better served by dedicated inference stacks (vLLM, TGI) than by PEFT's own runtime.

## When to Use / When Not

**Use when:**
- You want to fine-tune an LLM or diffusion model on limited GPU memory (LoRA / QLoRA).
- You need many task-specific checkpoints and can't afford a full copy of the model per task.
- You want adapters that plug into the existing Transformers / Diffusers / TRL training and inference code with minimal glue.
- You're serving one base model with several swappable specializations.

**Avoid when:**
- You have the compute and data for a full fine-tune and need every last point of quality — full fine-tuning is still the ceiling.
- Your model architecture is exotic and you're unwilling to work out `target_modules` and validate that adapters actually train.
- You need a turnkey, opinionated training pipeline (dataset handling, config, multi-GPU orchestration) — PEFT is the adapter layer, not the trainer; reach for a higher-level wrapper.
- You're on non-CUDA hardware and depend specifically on the 4-bit QLoRA path.

## Alternatives

- unslothai/unsloth — use instead when you want substantially faster, lower-memory LoRA/QLoRA on a single GPU via custom kernels, and can accept its supported-model list.
- hiyouga/LLaMA-Factory — use instead when you want a config/CLI/GUI-driven fine-tuning pipeline; it wraps PEFT rather than replacing the adapter math.
- axolotl-ai-cloud/axolotl — use instead when you want YAML-configured fine-tuning recipes over many models, again built on top of PEFT.
- huggingface/trl — use alongside PEFT (not instead) when you need SFT/DPO/PPO trainers; it accepts a `peft_config` directly.
- adapter-hub/adapters — use instead when you specifically want the AdapterHub research ecosystem (adapter fusion, hub sharing) tightly bound to Transformers.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2022-11 | Repository created; library grows out of LoRA + Transformers integration[^1]. |
| 0.3 | 2023-06 | Early releases stabilize LoRA, IA3, prefix/prompt/P-tuning; QLoRA lands in the ecosystem[^4]. |
| 0.7 | 2023-12 | Broader method coverage (LoHa, LoKr, OFT) and multi-adapter support mature. |
| 0.9 | 2024-03 | DoRA (weight-decomposed LoRA) added[^5]. |
| 0.10–0.13 | 2024 | VeRA, BOFT, X-LoRA and further methods; deeper Diffusers/Transformers integration. |
| 0.14+ | 2025 | Continued method additions and quantization/backend improvements. |

## References

[^1]: Hu et al., "LoRA: Low-Rank Adaptation of Large Language Models" — 2021. https://arxiv.org/abs/2106.09685
[^2]: PEFT documentation. https://huggingface.co/docs/peft
[^3]: PEFT adapter/method reference (supported PEFT methods). https://huggingface.co/docs/peft/index
[^4]: Dettmers et al., "QLoRA: Efficient Finetuning of Quantized LLMs" — 2023. https://arxiv.org/abs/2305.14314
[^5]: Liu et al., "DoRA: Weight-Decomposed Low-Rank Adaptation" — 2024. https://arxiv.org/abs/2402.09353

## Tags

python, pytorch, lora, qlora, fine-tuning, parameter-efficient, llm, adapters, huggingface, transformers, quantization, machine-learning
