# EleutherAI/gpt-neox

> EleutherAI's config-driven library for pretraining large autoregressive language models across many GPUs and many nodes.

[GitHub repo](https://github.com/EleutherAI/gpt-neox) ·
[Official website](https://www.eleuther.ai/) ·
[License: Apache-2.0](https://github.com/EleutherAI/gpt-neox/blob/main/LICENSE)

## Overview

GPT-NeoX is EleutherAI's framework for training large-scale, GPU-parallel autoregressive transformers. It began as the tooling behind GPT-NeoX-20B — one of the largest openly released dense language models at the time of its 2022 paper[^1] — and later trained the Pythia interpretability suite[^2]. Architecturally it is a fork-and-augment of NVIDIA's Megatron-LM (for tensor/pipeline model parallelism) wired to DeepSpeed (for ZeRO sharding and the distributed runtime), with a layer of EleutherAI configuration ergonomics and research features on top.

The critical thing to understand is scope. This is a *pretraining-from-scratch* codebase for people who have a GPU cluster and want to spend it. The README is blunt: if you are not training models with billions of parameters from scratch, this is the wrong tool, and you should use Hugging Face `transformers` for inference and most finetuning[^3]. GPT-NeoX does not ship a friendly high-level trainer API; it ships a launcher, a large YAML config surface, and Megatron-style parallelism that assumes multi-GPU as the baseline case.

Its defining tension is the DeepSpeed coupling. GPT-NeoX does not depend on stock DeepSpeed but on DeeperSpeed, EleutherAI's fork[^4]. Installing it can shadow or break other DeepSpeed-dependent packages in the same environment, so isolation is mandatory, not optional. In exchange you get a codebase that has been run at genuine supercomputer scale — ORNL Summit and Frontier, LUMI, AWS, CoreWeave — and supports launchers (Slurm, MPI, pdsh, IBM JSM) that most training libraries never touch.

## Getting Started

Environment isolation first — the DeeperSpeed dependency can break sibling packages:

```bash
git clone https://github.com/EleutherAI/gpt-neox
cd gpt-neox
pip install -r requirements/requirements.txt
python ./megatron/fused_kernels/setup.py install   # build fused CUDA kernels
```

A single-node training run is launched through `deepy.py`, the DeepSpeed-aware wrapper around the training entrypoint. Configs are composed by listing multiple YAML files; later files override earlier ones:

```bash
# Download and tokenize a small dataset, then train a tiny model on one node
python prepare_data.py -d ./data enwik8
python ./deepy.py train.py -d configs 125M.yml local_setup.yml
```

A config is plain YAML — model shape, parallelism degrees, optimizer, and data paths all live here rather than in code:

```yaml
# excerpt of a NeoX config
{
  "pipe_parallel_size": 1,
  "model_parallel_size": 1,
  "num_layers": 12,
  "hidden_size": 768,
  "num_attention_heads": 12,
  "train_micro_batch_size_per_gpu": 4,
  "optimizer": { "type": "Adam", "params": { "lr": 6.0e-4 } },
  "zero_optimization": { "stage": 1 }
}
```

## Architecture / How It Works

GPT-NeoX composes three layers that are easy to conflate:

1. **Megatron-LM lineage** — the transformer implementation and *intra-layer* model parallelism (tensor parallelism, set by `model_parallel_size`) descend from NVIDIA's Megatron-LM. This is what shards individual matmuls across GPUs.
2. **DeepSpeed / DeeperSpeed** — pipeline parallelism (`pipe_parallel_size`), ZeRO optimizer-state sharding, mixed precision, checkpointing, and the multi-node launcher come from DeepSpeed, accessed through EleutherAI's DeeperSpeed fork[^4].
3. **NeoX layer** — the YAML config system, `deepy.py` launcher, data preprocessing, Hugging Face export, and research features (rotary and ALiBi position embeddings, parallel attention+MLP blocks, Flash Attention, MoE, Mamba/RWKV variants, curriculum learning) are EleutherAI's own additions.

**3D parallelism** is the core idea: a training job is partitioned simultaneously along data (replicas), pipeline (layer groups across GPUs), and tensor (within-layer sharding) axes. The product of `pipe_parallel_size × model_parallel_size` must divide your GPU count, and the remainder becomes data-parallel width. Getting these three numbers right for a given model size and interconnect is the central skill of operating NeoX, and the docs give heuristics rather than a solver.

**Launcher abstraction.** `deepy.py` reads a hostfile (`node_ip slots=8` lines) and hands off to a multi-node runner — pdsh by default, or Slurm/MPI when configured. On exotic clusters (Summit's JSRun, LLNL Flux) you extend DeepSpeed's `multinode_runner.py` directly; the README documents this as a supported, if manual, path.

**Checkpoints** are DeepSpeed-format, sharded by parallelism layout, not Hugging Face format. A separate conversion script exports to HF `transformers` for downstream inference. This means a checkpoint's on-disk shape is tied to the parallelism degrees it was trained with — reloading with a different `model_parallel_size` requires conversion.

## Production Notes

**DeeperSpeed will fight your other packages.** Because the install pins EleutherAI's DeepSpeed fork, any other library in the same environment that imports `deepspeed` can break or silently get the wrong version. Use a dedicated conda env, container, or venv per NeoX project — the maintainers state this explicitly and it is the single most common setup failure.

**Version tiers matter for reproduction.** The 2023-03 v2.0 release moved onto modern upstream DeepSpeed; v1.0 preserves the older DeeperSpeed (based on DeepSpeed 0.3.15) that GPT-NeoX-20B and the original Pythia suite were trained on[^5]. If you are reproducing those specific models bit-for-bit, you likely need the v1.0 line, not `main`.

**Version pinning is tight.** The codebase has primarily been developed and tested against specific Python (3.8–3.10 historically) and PyTorch ranges, plus a specific Flash Attention major version (Flash Attention 2.x; 0.x/1.x were deprecated in 2023). "Other versions may work" is a real caveat, not boilerplate — CUDA, PyTorch, DeepSpeed, and Flash Attention version skew is the usual source of cryptic kernel-build failures.

**Fused kernels compile at install/first-run.** JIT fused-kernel compilation supports both NVIDIA and AMD (MI100, MI250X) GPUs, but a mismatched compiler/CUDA toolkit surfaces here first. Pre-building via `megatron.fused_kernels.load()` avoids a stall at job launch.

**Parallelism tuning is empirical.** There is no autotuner for the 3D split. Too much tensor parallelism across slow interconnect wrecks throughput; too little and large layers OOM. Expect a round of profiling per new cluster/model-size combination. Flash Attention and Transformer Engine give real speedups but only on the GPU architectures they target (Ampere/Hopper).

**This is research code, actively maintained but not a stable product API.** With ~7.4k stars, 1.1k forks, and commits through mid-2026, it is alive and used across academic, industry, and government labs. But configs, defaults, and supported feature combinations shift between versions, and many features (MoE, Mamba, RWKV, preference learning via DPO/KTO) landed incrementally and carry sharper edges than the core dense-transformer path.

## When to Use / When Not

**Use when:**
- You are pretraining a billion-plus-parameter model from scratch on a multi-GPU or multi-node cluster.
- You need 3D parallelism (tensor + pipeline + ZeRO data) and exotic launchers (Slurm, MPI, JSRun) that HPC schedulers require.
- You want a battle-tested path that has run on Summit, Frontier, LUMI, and major clouds.
- You are reproducing or extending EleutherAI models (Pythia, GPT-NeoX-20B) and want config parity.

**Avoid when:**
- You only need inference or serving — use huggingface/transformers, which supports GPT-NeoX architectures directly.
- You are finetuning an existing model on a single GPU — a lighter stack (PEFT, Axolotl, TRL) is far less operationally heavy.
- You cannot give it an isolated environment — the DeeperSpeed dependency makes shared envs risky.
- You want a stable, documented high-level trainer API rather than a YAML-plus-launcher research codebase.

## Alternatives

- NVIDIA/Megatron-LM — the upstream tensor/pipeline-parallel core NeoX descends from; use it directly when you want NVIDIA-maintained internals and don't need NeoX's config ergonomics or launcher breadth.
- microsoft/DeepSpeed — the ZeRO and parallelism engine underneath; use it when you're wiring parallelism into your own training loop rather than adopting a full framework.
- huggingface/nanotron — HF's minimalist 3D-parallel pretraining library; use when you want a smaller, more readable codebase over NeoX's accumulated surface area.
- pytorch/torchtitan — PyTorch-native reference for large-scale pretraining (FSDP2, tensor parallel); use when you want to stay inside first-party PyTorch abstractions.
- huggingface/transformers — not a trainer at NeoX's scale, but the correct choice for inference and most finetuning of GPT-NeoX-class models.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-12 | Repository created; early Megatron+DeepSpeed training stack. |
| GPT-NeoX-20B | 2022-02 | 20B-parameter dense model trained with the library; paper released April 2022[^1]. |
| Pythia | 2023 | Pythia interpretability model suite trained on the v1.0 line[^2]. |
| v1.0 | 2023 | Snapshot on legacy DeeperSpeed (DeepSpeed 0.3.15) — the 20B/Pythia baseline[^5]. |
| v2.0.0 | 2023-03-09 | Rebuilt on modern upstream DeepSpeed; the maintained line going forward[^5]. |
| ongoing | 2024–2026 | Incremental additions: MoE, Mamba/RWKV, AMD MI250X, Transformer Engine, DPO/KTO preference learning[^3]. |

## References

[^1]: Sid Black et al., "GPT-NeoX-20B: An Open-Source Autoregressive Language Model" — 2022. https://arxiv.org/abs/2204.06745
[^2]: Stella Biderman et al., "Pythia: A Suite for Analyzing Large Language Models Across Training and Scaling" — 2023. https://arxiv.org/abs/2304.01373
[^3]: GPT-NeoX README (features, news, scope guidance). https://github.com/EleutherAI/gpt-neox
[^4]: DeeperSpeed — EleutherAI's fork of DeepSpeed used by GPT-NeoX. https://github.com/EleutherAI/DeeperSpeed
[^5]: GPT-NeoX v2.0 release notes and versioning (v1.0 vs v2.0 DeepSpeed lineage). https://github.com/EleutherAI/gpt-neox/releases/tag/v2.0

## Tags

python, machine-learning, large-language-models, distributed-training, gpu, model-parallelism, deepspeed, megatron, pretraining, transformers, eleutherai
