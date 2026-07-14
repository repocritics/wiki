# huggingface/trl

> Post-training library for language models — SFT, reward modeling, and the preference/RL methods (DPO, GRPO, KTO, PPO) built on top of Hugging Face Transformers.

[GitHub repo](https://github.com/huggingface/trl) ·
[Official website](https://huggingface.co/docs/trl) ·
[License: Apache-2.0](https://github.com/huggingface/trl/blob/main/LICENSE)

## Overview

TRL ("Transformer Reinforcement Learning") is Hugging Face's library for the *post-training* stage of the LLM pipeline: everything that happens after pretraining a base model — supervised fine-tuning (SFT), reward modeling, and preference-optimization / RL methods like DPO, GRPO, KTO, and PPO[^1]. It began in 2020 as Leandro von Werra's `lvwerra/trl`, a compact PPO implementation for fine-tuning GPT-2 against a reward model[^2], and was folded into the Hugging Face organization as the RLHF wave (InstructGPT, then open Llama-era alignment) made post-training a mainstream need.

The defining design choice is that every TRL trainer is a thin subclass of the Transformers `Trainer`. You get Transformers' training loop, checkpointing, logging, and — through Accelerate — distributed execution (DDP, DeepSpeed ZeRO, FSDP) essentially for free, and TRL only adds the loss and data-collation logic specific to each method. This makes TRL easy to adopt if you already live in the Hugging Face stack, and awkward if you don't: it inherits both the ergonomics and the sharp edges of `transformers`, and its own API has changed often because the field it tracks changes often.

The center of gravity has shifted over the library's life. The original selling point was `PPOTrainer`; by 2023 the practical default became `DPOTrainer` (offline preference optimization, no reward model or sampling loop), and since DeepSeek-R1 the attention has moved to `GRPOTrainer` for reasoning-style RL with verifiable rewards. TRL is not a from-scratch RL framework — it is a convenience layer that keeps pace with whichever alignment method is currently winning.

## Getting Started

```bash
pip install trl
# or, for the latest unreleased trainers:
pip install git+https://github.com/huggingface/trl.git
```

Supervised fine-tuning on a chat dataset:

```python
from datasets import load_dataset
from trl import SFTTrainer

dataset = load_dataset("trl-lib/Capybara", split="train")

trainer = SFTTrainer(
    model="Qwen/Qwen2.5-0.5B",
    train_dataset=dataset,
)
trainer.train()
```

GRPO with a programmatic reward function (the DeepSeek-R1-style recipe):

```python
from datasets import load_dataset
from trl import GRPOTrainer
from trl.rewards import accuracy_reward

dataset = load_dataset("trl-lib/DeepMath-103K", split="train")

trainer = GRPOTrainer(
    model="Qwen/Qwen2.5-0.5B-Instruct",
    reward_funcs=accuracy_reward,   # or your own def f(completions, **kwargs) -> list[float]
    train_dataset=dataset,
)
trainer.train()
```

The `trl` CLI wraps the common trainers so you can run SFT/DPO/KTO without writing a script (`trl sft --model_name_or_path ... --dataset_name ...`).

## Architecture / How It Works

Each method is a `*Trainer` + `*Config` pair, where the config subclasses Transformers' `TrainingArguments`. The main families:

- **`SFTTrainer`** — next-token cross-entropy on completions. Handles chat-template formatting, prompt/completion masking (train only on the assistant turn), and packing of short sequences.
- **`DPOTrainer`** — Direct Preference Optimization[^3]. Offline, no sampling: given `(prompt, chosen, rejected)` triples it optimizes a closed-form preference loss against a frozen *reference* policy. The reference model is a second forward pass; with PEFT it is realized "for free" by disabling the LoRA adapter rather than loading a second copy. Related offline variants — CPO, ORPO, IPO — share most of this machinery.
- **`GRPOTrainer`** — Group Relative Policy Optimization from the DeepSeekMath paper[^4], the method behind DeepSeek-R1. For each prompt it samples a *group* of completions, scores them with reward function(s), and uses the group-normalized advantage (mean/variance within the group) instead of a learned value/critic network — which is what makes it lighter than PPO.
- **`KTOTrainer`** — Kahneman-Tversky Optimization[^5]. Aligns from unpaired binary (desirable/undesirable) signals rather than paired preferences.
- **`RewardTrainer`** — trains a scalar reward model on preference pairs (the classic RLHF step 2).
- **`PPOTrainer`** — the original online RL loop (policy + value head + reference + reward model). Still present but no longer the recommended default for most users.

**Online RL and generation.** For GRPO/PPO the throughput bottleneck is *generation*, not the gradient step. TRL integrates vLLM for fast sampling, in two modes: a colocated mode (vLLM shares the training GPUs) and a server mode (`trl vllm-serve` runs generation as a separate process the trainer calls). Weights are synced from the trainer to the vLLM engine each step.

**Scaling.** All distribution is delegated to Accelerate, so multi-GPU/multi-node is configured through `accelerate config` / DeepSpeed / FSDP rather than anything TRL-specific. PEFT (LoRA/QLoRA) and Unsloth kernels plug in for memory-constrained training.

## Production Notes

**API instability is the number-one operational cost.** TRL is pre-1.0 and tracks a fast-moving field; minor releases regularly rename or move arguments. Concrete examples seen across versions: dataset-formatting args migrating out of the trainer into `SFTConfig`, the `tokenizer=` argument being replaced by `processing_class=` (a Transformers-wide change), and reward/collator signatures evolving. Pin an exact `trl` version in production and read the release notes before upgrading — code from a blog post six months old frequently will not run unmodified.

**Version coupling with `transformers`.** Because trainers subclass the Transformers `Trainer`, a `trl` upgrade often forces a `transformers` (and sometimes `accelerate`/`peft`) upgrade. Treat the four as a single version set.

**Dataset format is a common footgun.** TRL distinguishes "conversational" format (lists of `{"role", "content"}` messages) from "standard" format (plain `prompt`/`completion`/`chosen`/`rejected` text fields), and different trainers expect different shapes. A silently mis-shaped dataset tends to fail late or train on the wrong tokens rather than error early. Verify that prompt masking is doing what you expect before a long run.

**Memory.** DPO/PPO conceptually need a reference model resident alongside the policy; without PEFT that roughly doubles model memory. GRPO adds the vLLM engine's KV cache and weights on top of the trainer's — colocated mode can OOM on the same GPUs that trained fine under SFT. Budget for generation memory separately.

**Reward functions (GRPO).** Rewards are ordinary Python callables `(completions, **kwargs) -> list[float]`, which is flexible but means correctness, batching, and latency of your reward code are on you. A slow reward function (e.g. calling out to a verifier or a judge model) can dominate step time even with vLLM generation.

**Reproducibility of RL runs.** Online methods (GRPO/PPO) are sensitive to sampling temperature, group size, KL coefficient, and reward scaling; small config differences produce large behavioral differences. Log everything and expect to sweep.

## When to Use / When Not

**Use when:**
- You already use Hugging Face Transformers/Datasets/PEFT and want SFT, DPO, or GRPO without building the loss and data plumbing yourself.
- You want to reproduce a published recipe (DeepSeek-R1-style GRPO, DPO alignment) with a maintained implementation.
- You need PEFT/QLoRA fine-tuning on modest hardware and value a batteries-included trainer.

**Avoid when:**
- You need a large-scale, Ray-based, disaggregated RLHF system (dedicated actor/rollout/reward clusters) — TRL's Accelerate-based model is simpler but less horizontally scalable than purpose-built frameworks.
- You want a stable, rarely-changing API to build a long-lived product on — TRL iterates fast and breaks compatibility.
- You are outside the Hugging Face ecosystem and don't want to inherit the `transformers` training stack.
- You just need config-driven fine-tuning without touching Python — a wrapper like Axolotl may fit better.

## Alternatives

- OpenRLHF/OpenRLHF — Ray + vLLM + DeepSpeed RLHF framework; use when you need to scale PPO/GRPO across many nodes with disaggregated actors.
- volcengine/verl — HybridFlow RL library aimed at large-scale, high-throughput reasoning RL; use when TRL's single-controller model is the bottleneck.
- axolotl-ai-cloud/axolotl — YAML-config fine-tuning frontend (often over TRL/Transformers); use when you want SFT/DPO without writing training code.
- unslothai/unsloth — memory- and speed-optimized single-GPU fine-tuning; use when you're constrained to one consumer GPU. TRL can call its kernels.
- huggingface/peft — the LoRA/QLoRA layer TRL uses; complementary, not a replacement.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020 | `lvwerra/trl` — PPO fine-tuning of GPT-2 against a reward model[^2]. |
| — | 2023 | Moved under the `huggingface` org; `SFTTrainer` and `DPOTrainer` added as the library refocuses on RLHF/preference tuning[^3]. |
| — | 2024 | `KTOTrainer`, `ORPO`, and other offline preference variants added; Accelerate/PEFT/DeepSpeed scaling matured. |
| — | 2025 | `GRPOTrainer` added following DeepSeek-R1; vLLM generation integration for online RL. |
| current | 2026 | Multi-environment agentic RL for GRPO (Harbor / OpenEnv); `KTOTrainer` graduated to stable API. Still pre-1.0. |

## References

[^1]: TRL documentation — index. https://huggingface.co/docs/trl/index
[^2]: `lvwerra/trl` original PPO implementation and TRL citation entry (von Werra et al., 2020). https://github.com/huggingface/trl
[^3]: Rafailov et al., "Direct Preference Optimization: Your Language Model is Secretly a Reward Model" — arXiv:2305.18290. https://huggingface.co/papers/2305.18290
[^4]: Shao et al., "DeepSeekMath" (introduces GRPO) — arXiv:2402.03300. https://huggingface.co/papers/2402.03300
[^5]: Ethayarajh et al., "KTO: Model Alignment as Prospect Theoretic Optimization" — arXiv:2402.01306. https://huggingface.co/papers/2402.01306

## Tags

python, llm, rlhf, fine-tuning, reinforcement-learning, dpo, grpo, post-training, huggingface, transformers, preference-optimization, machine-learning
