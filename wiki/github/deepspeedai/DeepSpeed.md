# deepspeedai/DeepSpeed

> PyTorch training/inference optimization library whose ZeRO family lets you train models larger than a single GPU's memory.

[GitHub repo](https://github.com/deepspeedai/DeepSpeed) ·
[Official website](https://www.deepspeed.ai/) ·
[License: Apache-2.0](https://github.com/deepspeedai/DeepSpeed/blob/master/LICENSE)

## Overview

DeepSpeed is a deep-learning optimization library layered on top of PyTorch. Its reason to exist is a single memory problem: a model's parameters, gradients, and optimizer states (for Adam, roughly 16 bytes per parameter in mixed precision) do not fit on one accelerator once the model is large. DeepSpeed's answer is **ZeRO** (Zero Redundancy Optimizer)[^1], which shards those states across the data-parallel group instead of replicating them, so aggregate GPU memory scales with the number of devices rather than being capped by one.

The project came out of Microsoft's "AI at Scale" effort and was the training substrate behind several of the largest language models of their era, including Megatron-Turing NLG (530B) and BLOOM (176B)[^2]. In 2025 it moved out of the `microsoft/` GitHub organization to a vendor-neutral `deepspeedai/` org — old `microsoft/DeepSpeed` URLs now redirect here[^3]. The core team (Rajbhandari, Rasley, Ruwase, He, and others) has substantially dispersed to other companies, and much frontier serving work has shifted to inference-specialized stacks; DeepSpeed today is most valued as a mature, config-driven training scaler rather than a serving engine.

The defining tension: DeepSpeed buys you scale by trading compute-locality for communication. ZeRO-3 and offload move memory pressure onto the interconnect and PCIe/NVMe bus. On fast NVLink/InfiniBand fabrics that trade is nearly free; on commodity hardware it can dominate step time. Understanding which knob costs which bandwidth is the whole job.

## Getting Started

```bash
pip install deepspeed
ds_report   # prints which C++/CUDA ops your machine can build, and CUDA/torch versions
```

A `ds_config.json` describes the optimization strategy rather than hard-coding it in Python:

```json
{
  "train_micro_batch_size_per_gpu": 4,
  "gradient_accumulation_steps": 8,
  "bf16": { "enabled": true },
  "zero_optimization": {
    "stage": 3,
    "offload_optimizer": { "device": "cpu" }
  }
}
```

```python
import deepspeed

model_engine, optimizer, _, _ = deepspeed.initialize(
    model=model,
    model_parameters=model.parameters(),
    config="ds_config.json",
)

for batch in data_loader:
    loss = model_engine(batch)      # forward
    model_engine.backward(loss)     # DeepSpeed handles grad partitioning/accumulation
    model_engine.step()             # optimizer + zero_grad
```

```bash
deepspeed --num_gpus 8 train.py   # the launcher sets up the distributed process group
```

## Architecture / How It Works

**ZeRO stages** partition progressively more state across the data-parallel group[^1]:
- **Stage 1** — optimizer states only.
- **Stage 2** — optimizer states + gradients.
- **Stage 3** — optimizer states + gradients + parameters. Each layer's full weights are gathered (all-gather) just before its forward/backward and released afterward, so peak memory is roughly one layer's worth rather than the whole model.

**Offload** extends this beyond GPU memory. ZeRO-Offload pushes optimizer state and computation to CPU; **ZeRO-Infinity** additionally offloads to NVMe, letting a single node touch models it cannot come close to holding in GPU RAM[^4]. The cost is bandwidth: offload only performs well with fast CPU-GPU links and, for Infinity, high-throughput NVMe.

**3D parallelism** composes data parallelism (ZeRO) with tensor/model parallelism and pipeline parallelism, typically alongside Megatron for the tensor-parallel piece. Beyond that, DeepSpeed ships MoE training (expert parallelism), Ulysses sequence parallelism for long context, and activation checkpointing.

**Ops.** Performance-critical pieces (fused Adam, transformer kernels, quantization) are C++/CUDA/HIP extensions. By default they are **JIT-compiled** at first use via PyTorch's ninja-based extension loader, which means the machine needs a matching `nvcc`/`hipcc` toolchain at runtime. Ops can instead be pre-built at install time with `DS_BUILD_*` environment flags.

The library is deliberately configuration-first: the `deepspeed.initialize` call plus a JSON file is the entire integration surface, which is why HuggingFace Transformers, Accelerate, PyTorch Lightning, and others wrap it rather than reimplementing sharding.

## Production Notes

- **Op build failures are the #1 onboarding pain.** JIT compilation needs the CUDA toolkit `nvcc` version to line up with the CUDA that PyTorch was built against. Mismatches produce cryptic ninja/compiler errors. Run `ds_report` first; pin toolchains in the container; pre-build ops (`DS_BUILD_FUSED_ADAM=1` etc.) for reproducible images.
- **ZeRO-3 is communication-bound.** The per-layer all-gather/release means a slow interconnect (PCIe-only, no NVLink, Ethernet instead of InfiniBand) can make Stage 3 dramatically slower than Stage 2. Small models on few GPUs often run *worse* under Stage 3 than plain DDP.
- **Checkpoints are sharded.** ZeRO writes partitioned checkpoints, not a single consolidated state dict. To load weights into a non-DeepSpeed runtime (inference, HF `from_pretrained`) you must consolidate with the bundled `zero_to_fp32.py` script. Forgetting this is a common "my checkpoint won't load" bug.
- **Batch-size arithmetic is enforced.** `train_batch_size` must equal `train_micro_batch_size_per_gpu × gradient_accumulation_steps × world_size`; DeepSpeed errors if they are inconsistent, and it is easy to get wrong when changing GPU count.
- **Offload can look like a hang.** With NVMe offload, step time is gated by disk throughput; a "stuck" run is frequently just I/O-bound. Measure NVMe bandwidth before assuming Infinity will help.
- **Inference has been de-emphasized.** DeepSpeed-Inference / MII exist, but for LLM serving most teams have moved to vLLM or TensorRT-LLM; treat DeepSpeed primarily as a *training* tool in 2026 unless you have a specific reason.
- **Version coupling.** DeepSpeed tracks PyTorch closely; upgrading torch can require a matching DeepSpeed bump and an op rebuild.

## When to Use / When Not

**Use when:**
- Your model plus Adam optimizer states do not fit on one GPU and you have multiple accelerators to shard across.
- You want to push a large model onto limited hardware via CPU/NVMe offload and can accept the bandwidth cost.
- You need MoE, pipeline, or long-sequence (Ulysses) parallelism composed with data parallelism.
- You are already inside HuggingFace/Lightning and want to enable sharding through a config file.

**Avoid when:**
- The model comfortably fits with plain DDP — ZeRO's communication overhead is pure loss there.
- You want to stay in the PyTorch tree with no extra native deps — FSDP covers most ZeRO-3 use cases.
- Your goal is high-throughput inference serving — reach for vLLM or TensorRT-LLM.
- You cannot control the build toolchain (locked-down environment with no matching `nvcc`).

## Alternatives

- pytorch/pytorch — native **FSDP** implements ZeRO-3-style parameter sharding in-tree; use it when you want no third-party dependency and standard PyTorch tooling.
- NVIDIA/Megatron-LM — use when you need mature tensor/pipeline parallelism for very large transformers on NVIDIA hardware (often combined with, not against, DeepSpeed).
- huggingface/accelerate — use when you want one abstraction that can drive either DeepSpeed or FSDP without committing to either API.
- hpcaitech/ColossalAI — use when you want an alternative all-in-one large-model training system with its own sharding/offload stack.
- vllm-project/vllm — use instead of DeepSpeed-Inference when the task is LLM serving rather than training.

## History

| Version | Date | Notes |
|---------|------|-------|
| ZeRO paper | 2019-10 | Rajbhandari et al., memory optimizations toward trillion-parameter models[^1]. |
| v0.1 | 2020-02 | Initial open-source release under `microsoft/DeepSpeed`. |
| ZeRO-Offload | 2021 | Optimizer state + compute offloaded to CPU[^4]. |
| ZeRO-Infinity | 2021 | NVMe offload; single-node extreme-scale training[^4]. |
| DeepSpeed-MoE | 2022 | Expert-parallel mixture-of-experts training and inference. |
| DeepSpeed-Chat | 2023 | End-to-end RLHF training pipeline. |
| org move | 2025 | Repo relocated to vendor-neutral `deepspeedai/` org[^3]. |

## References

[^1]: Rajbhandari, Rasley, Ruwase, He. "ZeRO: Memory Optimizations Toward Training Trillion Parameter Models" — arXiv:1910.02054, SC '20. https://arxiv.org/abs/1910.02054
[^2]: DeepSpeed README, "DeepSpeed Adoption" — MT-NLG 530B, BLOOM 176B, and others. https://github.com/deepspeedai/DeepSpeed
[^3]: Repository now resolves to `deepspeedai/DeepSpeed`; `microsoft/DeepSpeed` redirects. https://github.com/deepspeedai/DeepSpeed
[^4]: Rajbhandari et al., "ZeRO-Infinity: Breaking the GPU Memory Wall for Extreme Scale Deep Learning" — arXiv:2104.07857. https://arxiv.org/abs/2104.07857

## Tags

python, pytorch, distributed-training, deep-learning, zero, model-parallelism, mixture-of-experts, gpu, machine-learning, optimization, llm-training
