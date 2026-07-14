# axolotl-ai-cloud/axolotl

> A YAML-configured wrapper over the Hugging Face training stack that turns LLM post-training (SFT, LoRA/QLoRA, DPO, GRPO) into a single config file and CLI command.

[GitHub repo](https://github.com/axolotl-ai-cloud/axolotl) ·
[Official docs](https://docs.axolotl.ai) ·
[License: Apache-2.0](https://github.com/axolotl-ai-cloud/axolotl/blob/main/LICENSE)

## Overview

Axolotl is an open-source framework for fine-tuning and post-training large language models. It began in April 2023 under the OpenAccess-AI-Collective organization and later moved to the `axolotl-ai-cloud` org backed by Axolotl AI, the commercial entity that funds development[^1]. Its core proposition is uniformity: instead of writing bespoke training scripts against `transformers`, `peft`, `trl`, and `accelerate`, you describe the model, dataset, adapter, and distributed strategy in one YAML file and run `axolotl train config.yml`. The same config drives preprocessing, training, evaluation, quantization, and inference.

The audience is practitioners who fine-tune frequently across many models and methods and want a stable config surface rather than a library API. Axolotl's real value is integration breadth — it wires together a large and fast-moving set of techniques (multipacking, Flash Attention 2/3/4, Liger kernels, FSDP1/FSDP2, DeepSpeed, sequence/context/tensor/expert parallelism, GRPO, QAT down to NVFP4) so that turning one on is a config key rather than a dependency-resolution project[^2].

The defining tension is exactly that breadth. Axolotl is a thin-to-medium abstraction over a stack that changes weekly, and it tracks new models and kernels aggressively — the changelog shows near-monthly additions of new architectures and optimizers[^2]. That currency is the reason to use it, but it also means the config schema, defaults, and dependency pins are moving targets, and a working setup is tightly bound to specific versions of PyTorch, CUDA, and the transformers ecosystem underneath.

## Getting Started

```bash
# uv is the first-class installer as of 2026
curl -LsSf https://astral.sh/uv/install.sh | sh
export UV_TORCH_BACKEND=cu130          # match your CUDA
uv venv --python 3.12 && source .venv/bin/activate
uv pip install torch==2.12.0 torchvision
uv pip install --no-build-isolation axolotl[deepspeed]

axolotl fetch examples                 # download example configs
axolotl train examples/llama-3/lora-1b.yml
```

A minimal LoRA config is a single YAML file:

```yaml
base_model: NousResearch/Meta-Llama-3-8B
load_in_4bit: true
adapter: qlora
datasets:
  - path: mhenrichsen/alpaca_2k_test
    type: alpaca
sequence_len: 2048
sample_packing: true
micro_batch_size: 2
gradient_accumulation_steps: 4
num_epochs: 1
learning_rate: 0.0002
flash_attention: true
output_dir: ./outputs/qlora-out
```

Docker images (`axolotlai/axolotl:main-latest`) avoid local dependency resolution and are often the less error-prone path.

## Architecture / How It Works

Axolotl is a configuration-and-orchestration layer, not a training engine. The actual forward/backward passes, optimizers, and distributed sharding come from PyTorch, `transformers`, `peft`, `trl`, `accelerate`/DeepSpeed, and a set of custom or vendored CUDA/Triton kernels. Axolotl's own code is largely: a Pydantic-validated config schema, dataset loaders and prompt formatters, a monkeypatch layer that swaps attention implementations and loss kernels into upstream model classes, and CLI plumbing (`preprocess`, `train`, `evaluate`, `inference`, `quantize`).

The config file is the whole interface. It is validated against a schema (`axolotl config-schema` dumps it) and expanded into training arguments. This is why a config that worked six months ago may fail today — a field may have been renamed, a default flipped, or an integration's peer dependency bumped. The framework's power (flip `sequence_parallel: true` or `quantize_moe_experts: true` and get behavior that would otherwise be days of integration work) is inseparable from this fragility.

Distributed training is delegated: FSDP1/FSDP2 and DeepSpeed handle sharding, and Axolotl composes ND parallelism (context, tensor, and expert parallelism alongside FSDP) on top[^3]. Performance features like multipacking (sample packing that concatenates short sequences to reduce padding waste) and the various fused-attention/fused-loss kernels are the reason people accept the operational overhead — they materially cut VRAM and wall-clock versus a naive `transformers` Trainer loop.

Axolotl also ships agent-oriented documentation bundled in the pip package (`axolotl agent-docs`, `axolotl config-schema`), an unusual and useful feature for driving the tool from coding agents without cloning the repo.

## Production Notes

- **Dependency resolution is the main footgun.** Axolotl pins to specific PyTorch/CUDA combinations, and `--no-build-isolation` installs (for Flash Attention, DeepSpeed, custom kernels) compile against your local toolchain. Version drift between torch, CUDA, `flash-attn`, and `transformers` is the most common source of broken environments. The Docker images exist specifically to sidestep this; use them in CI and on ephemeral cloud GPUs.
- **Configs are version-bound.** There is no long-term config stability guarantee. Pin the Axolotl version alongside your configs, and expect to revisit them on upgrade. Reproducing an old run means reproducing the old environment, not just the YAML.
- **Hardware assumptions are real.** `bf16` and Flash Attention effectively require an Ampere-or-newer NVIDIA GPU; AMD is supported but a smaller, less-trodden path. Newer quantization formats (NVFP4/MXFP4, MoE expert quantization) target recent datacenter GPUs.
- **Multipacking changes loss semantics.** Sample packing improves throughput but interacts with attention masking and loss masking; misconfiguration silently trains on the wrong tokens (a recurring class of bug, e.g. multimodal assistant-only loss masking has needed fixes)[^2]. Validate that your effective batch and masking are what you intend.
- **Telemetry is opt-out.** Basic usage telemetry is on by default; set `AXOLOTL_DO_NOT_TRACK=1` to disable it. Note this for regulated or air-gapped environments.
- **Preprocess separately for large datasets.** Run `axolotl preprocess` to cache tokenized/packed data before multi-GPU training, otherwise every rank redoes the work.

## When to Use / When Not

**Use when:**
- You fine-tune many different models/methods and want one config-driven workflow instead of per-project training scripts.
- You need modern distributed or memory techniques (FSDP2, ND parallelism, QAT, GRPO, Liger/Flash kernels) without integrating them yourself.
- You are comfortable pinning versions and running in Docker to keep environments reproducible.

**Avoid when:**
- You want a stable, slow-moving API to build a product on — the config surface and dependency stack move too fast.
- You need fine-grained programmatic control over the training loop; use the underlying libraries (`trl`, `transformers`) directly.
- You are doing single-GPU LoRA/QLoRA and care most about raw speed/VRAM — a more specialized tool may be leaner.
- Your environment can't accommodate compiling CUDA kernels or a specific torch/CUDA pin.

## Alternatives

- huggingface/trl — the RLHF/post-training library Axolotl builds on; use it directly when you want Python-level control instead of a YAML wrapper.
- hiyouga/LLaMA-Factory — comparable config/GUI-driven fine-tuning framework with broad model support; use it if you prefer a web UI and a different config dialect.
- unslothai/unsloth — hand-optimized single-GPU LoRA/QLoRA kernels; use it when single-GPU speed and VRAM are the priority over multi-node breadth.
- pytorch/torchtune — official PyTorch recipe-based fine-tuning library; use it when you want a leaner, PyTorch-native stack with fewer moving integrations.
- modelscope/ms-swift — Alibaba's fine-tuning/inference toolkit; use it if you live in the ModelScope ecosystem or need its model coverage.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2023-04 | First release under OpenAccess-AI-Collective[^1]. |
| Org move | ~2024 | Repo moved to `axolotl-ai-cloud` (Axolotl AI)[^1]. |
| — | 2025-02 | GRPO support and LoRA memory/speed optimizations added[^2]. |
| — | 2025-03 | Sequence Parallelism and beta multimodal (VLM) fine-tuning[^2]. |
| — | 2025-07 | ND Parallelism (compose CP/TP/FSDP) landed[^3]. |
| — | 2026-04 | uv-first install, Async GRPO, Flash Attention 4 integration[^2]. |
| — | 2026-07 | NVFP4 (4-bit) MoE LoRA via ScatterMoE/SonicMoE[^2]. |

As of mid-2026 the project is actively developed — roughly 12.2k stars, 1.4k forks, and daily pushes — with a large contributor base and a commercial sponsor behind it[^4].

## References

[^1]: Axolotl repository and organization — `axolotl-ai-cloud/axolotl`, created 2023-04-14, previously under OpenAccess-AI-Collective. https://github.com/axolotl-ai-cloud/axolotl
[^2]: Axolotl README "Latest Updates" changelog (GRPO, LoRA optims, multimodal, kernels, NVFP4 MoE LoRA, loss-masking fixes). https://github.com/axolotl-ai-cloud/axolotl#readme
[^3]: "N-Dimensional Parallelism" — Axolotl docs and Hugging Face blog on composing CP/TP/FSDP. https://docs.axolotl.ai/docs/nd_parallelism.html
[^4]: GitHub repository metadata (stars, forks, last push), fetched 2026-07-15. https://github.com/axolotl-ai-cloud/axolotl

## Tags

python, llm, fine-tuning, lora, qlora, post-training, distributed-training, pytorch, machine-learning, yaml-config, huggingface
