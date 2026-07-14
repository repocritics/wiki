# huggingface/accelerate

> A thin wrapper that runs a raw PyTorch training loop unchanged across CPU, GPU, multi-GPU, TPU, and sharded (FSDP/DeepSpeed) configurations.

[GitHub repo](https://github.com/huggingface/accelerate) ·
[Official docs](https://huggingface.co/docs/accelerate) ·
[License: Apache-2.0](https://github.com/huggingface/accelerate/blob/main/LICENSE)

## Overview

Accelerate is Hugging Face's device- and distribution-abstraction layer for
PyTorch. First released in 2021[^1], its pitch is deliberately narrow: keep your
own training loop, add about five lines, and the same script runs on a laptop
CPU, a single GPU, eight GPUs across two nodes, a TPU pod, or under DeepSpeed
ZeRO — without rewriting the loop. It is explicitly *not* a high-level trainer;
the README states outright that if you don't want to write a training loop
yourself, you should use something else[^2].

Its real importance is as infrastructure rather than an end-user tool. Accelerate
is a required dependency of `transformers`, and is the distributed backend under
the `transformers` `Trainer`, `diffusers`, TRL, and PEFT[^3]. Most people who run
Accelerate have never imported it directly — they hit it through
`from_pretrained(device_map="auto")` or a `Trainer.train()` call. That dual role
(minimal library for loop-writers, plus load-bearing plumbing for the entire HF
stack) is the defining tension: the "5 lines" surface is genuinely small, but the
configuration surface underneath it — FSDP, DeepSpeed, Megatron-LM, fp8,
big-model offloading — is large and leaks through as soon as you scale past one
GPU.

## Getting Started

```bash
pip install accelerate        # tested on Python 3.9+ and PyTorch 2.0+
accelerate config             # interactive: writes ~/.cache/huggingface/accelerate/default_config.yaml
```

```python
from accelerate import Accelerator
import torch, torch.nn.functional as F

accelerator = Accelerator()               # reads env / config, picks backend
model = torch.nn.Transformer()
optimizer = torch.optim.Adam(model.parameters())
data = torch.utils.data.DataLoader(dataset, shuffle=True)

# prepare() wraps each object for the active backend + device
model, optimizer, data = accelerator.prepare(model, optimizer, data)

model.train()
for source, targets in data:              # dataloader now shards across processes
    optimizer.zero_grad()
    loss = F.cross_entropy(model(source), targets)
    accelerator.backward(loss)            # replaces loss.backward(); handles scaling
    optimizer.step()
```

Launch with the CLI, which replaces `torchrun` / `xmp.spawn`:

```bash
accelerate launch --multi_gpu --num_processes 2 train.py
```

## Architecture / How It Works

The entire public API funnels through one object, `Accelerator`. `prepare()` is
where the abstraction lives: it inspects the configured backend and returns
wrapped versions of whatever you pass.

- **Model** — wrapped in the backend's parallel container: PyTorch DDP for plain
  multi-GPU, `FullyShardedDataParallel`, DeepSpeed's engine, or left bare for
  single-device.
- **DataLoader** — replaced with one that shards batches across processes (each
  rank sees a disjoint slice) and moves tensors to the device automatically. This
  is why effective batch size becomes `per_device_batch × num_processes ×
  grad_accum` — a frequent source of silent misconfiguration.
- **Optimizer** — wrapped as `AcceleratedOptimizer` so mixed-precision gradient
  scaling and gradient accumulation hook into `step()`.

`backward()` centralizes loss scaling (fp16/bf16/fp8) so the same loop works
regardless of precision. Precision itself is delegated: fp8 requires NVIDIA
Transformer Engine or MS-AMP and appropriate hardware, not an Accelerate
implementation[^2].

Accelerate does not implement the sharding algorithms. FSDP is PyTorch's,
DeepSpeed ZeRO is Microsoft's, Megatron-LM is NVIDIA's. Accelerate is the *config
glue*: `accelerate config` produces a YAML that selects a backend and its
options, and `accelerate launch` spawns the processes (wrapping
`torch.distributed.run` for GPUs, XLA spawn for TPUs, or `notebook_launcher` for
Colab/Kaggle). The library's value is a uniform surface over incompatible
launchers and parallelism strategies, not the strategies themselves.

A second, largely separate subsystem is **Big Model Inference**:
`init_empty_weights()` builds a model on the meta device (no memory), and
`load_checkpoint_and_dispatch()` / `device_map="auto"` distribute weights across
GPUs, CPU RAM, and disk with hooks that move layers on and off the accelerator
during the forward pass. This is the machinery `transformers` uses to load models
larger than a single GPU, and it has nothing to do with the training path — it
just ships in the same package.

## Production Notes

**The "5 lines" leak the moment you scale.** Correct multi-process training
requires several things the minimal example omits: `accelerator.clip_grad_norm_()`
instead of `torch.nn.utils.clip_grad_norm_`, passing the LR scheduler through
`prepare()` (or it steps the wrong number of times), `accelerator.unwrap_model()`
before saving, `set_epoch` on samplers for correct shuffling, and using
`accelerator.gather()` / `gather_for_metrics()` for evaluation so metrics aren't
computed on one shard. None of these are enforced; wrong code trains and produces
plausible-but-wrong results.

**Configuration precedence is confusing.** DeepSpeed and FSDP can be configured
three ways — a plugin object in Python, the `accelerate config` YAML, or (for
DeepSpeed) a separate JSON config with `"auto"` values that expect the launcher to
fill them in. When these disagree, the effective config is hard to predict.
DeepSpeed's `"auto"` fields in particular only resolve correctly when driven
through the `transformers` `Trainer`; hand-rolled loops must set them explicitly.

**Checkpoint portability is a real hazard.** Sharded FSDP/DeepSpeed checkpoints
are written per-rank and are sensitive to world size and PyTorch version.
Resuming on a different number of GPUs, or converting a sharded checkpoint to a
single consolidated file, is a known friction point and has changed across
PyTorch's FSDP1 → FSDP2 transition, which Accelerate has been tracking.

**Version coupling.** Accelerate is pinned tightly to both PyTorch and
`transformers`; because it's a hard dependency of `transformers`, a mismatched
Accelerate version tends to surface as opaque errors deep in someone else's stack
rather than as a clear version complaint. Upgrade the three together.

**Inference offloading is slow by default.** `device_map="auto"` will happily
offload layers to CPU or disk to make a model fit, and throughput then collapses
by orders of magnitude with no error — only a slow forward pass. Verify the
resulting `device_map` before assuming a model is "running on GPU."

**`notebook_launcher` and CUDA init.** Launching distributed training from a
notebook fails if CUDA has already been initialized in the parent process;
training setup must be deferred into the launched function.

## When to Use / When Not

**Use when:**
- You want to keep full control of your PyTorch training loop but need it to run
  unchanged across single-GPU, multi-GPU, multi-node, and TPU.
- You want one config surface over DDP, FSDP, and DeepSpeed instead of learning
  each launcher.
- You need to load or run a model that doesn't fit on one GPU (Big Model
  Inference / `device_map`).
- You're building a library on top of PyTorch and want distribution handled
  without imposing a framework on your users.

**Avoid when:**
- You don't want to write the loop at all — use a high-level trainer (Lightning,
  the `transformers` Trainer, Composer) instead.
- You need a single-vendor, deeply-optimized ZeRO/pipeline setup — using DeepSpeed
  or Megatron-LM directly gives more control than the abstraction layer.
- Your workload is pure large-scale data/experiment orchestration across a cluster
  — Ray Train addresses scheduling concerns Accelerate does not.

## Alternatives

- pytorch/pytorch — raw `DistributedDataParallel` + `torchrun`. No abstraction,
  full control; use directly when you only target one topology and want zero deps.
- Lightning-AI/pytorch-lightning — use instead when you want the loop, callbacks,
  and logging owned for you rather than writing them.
- microsoft/DeepSpeed — use directly when ZeRO-3, offload, and pipeline
  parallelism are the point and you want its full knob set, not a curated subset.
- ray-project/ray — use Ray Train when the hard problem is cluster scheduling,
  fault tolerance, and heterogeneous orchestration, not loop portability.
- mosaicml/composer — use when you want an efficiency-focused trainer with
  built-in speedup methods rather than a minimal wrapper.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021-04 | Initial release: `Accelerator`, `prepare()`, `accelerate launch`[^1]. |
| 0.3–0.5 | 2021 | DeepSpeed (experimental) and TPU support added. |
| 0.6–0.9 | 2022 | Big Model Inference: `init_empty_weights`, `device_map`, disk/CPU offload. |
| 0.11 | 2022 | FSDP integration (experimental). |
| 0.16 | 2023 | Megatron-LM integration. |
| 0.20+ | 2023 | fp8 via Transformer Engine / MS-AMP; broader FSDP support. |
| 1.0.0 | 2024 | First stable API contract; deprecations cleaned up[^4]. |

## References

[^1]: Accelerate releases and changelog. https://github.com/huggingface/accelerate/releases
[^2]: Accelerate README — "Why should I use / Why shouldn't I use" and supported integrations. https://github.com/huggingface/accelerate/blob/main/README.md
[^3]: Accelerate documentation index. https://huggingface.co/docs/accelerate
[^4]: "Accelerate 1.0.0" — Hugging Face blog / release notes (2024). https://huggingface.co/blog/accelerate-v1

## Tags

python, pytorch, distributed-training, machine-learning, gpu, deep-learning, fsdp, deepspeed, mixed-precision, huggingface, multi-gpu
