# Lightning-AI/pytorch-lightning

> A structural wrapper over PyTorch that removes training-loop boilerplate and scales the same model code from a laptop CPU to thousands of GPUs.

[GitHub repo](https://github.com/Lightning-AI/pytorch-lightning) ·
[Official website](https://lightning.ai/pytorch-lightning/) ·
[License: Apache-2.0](https://github.com/Lightning-AI/pytorch-lightning/blob/master/LICENSE)

## Overview

PyTorch Lightning is a high-level training framework on top of PyTorch. You subclass `LightningModule` and put your model, optimizer config, and per-batch step logic into named hooks (`training_step`, `validation_step`, `configure_optimizers`); a `Trainer` object then owns the actual loop — the `for epoch / for batch / zero_grad / backward / step` machinery, plus mixed precision, gradient accumulation, checkpointing, logging, and multi-device dispatch. The pitch is that the "science" (what the model computes) is decoupled from the "engineering" (how it runs), so the same `LightningModule` runs unchanged on CPU, one GPU, 8 GPUs, or 32 nodes by changing only `Trainer` flags[^1].

It was started by William Falcon and first published in 2019[^2], reaching a stable 1.0 in October 2020[^3]. The project is now maintained by Lightning AI (the company, formerly Grid.ai), which also sells a GPU cloud — the README and docs are threaded with referral links to that platform, though the framework itself is Apache-2.0 and runs anywhere. As of this writing the repo has ~31k stars and is actively maintained, with commits landing continuously on `master`.

The defining tension is abstraction versus transparency. Lightning is genuinely convenient for standard supervised-training shapes, but the loop it hides is large, and the hook ordering is something you must learn to debug non-trivial training. When your training fits the intended mold it saves real boilerplate; when it doesn't (custom optimization schedules, unusual multi-optimizer GANs, RL loops), you spend time fighting the framework's control flow instead of writing PyTorch. The `lightning.fabric` sub-package exists precisely as an escape hatch for that case (see Architecture).

## Getting Started

```bash
pip install lightning        # imports as `lightning` / `lightning.pytorch`
# or the standalone package:
pip install pytorch-lightning  # imports as `pytorch_lightning`
```

```python
import torch, torch.nn as nn, torch.nn.functional as F
import lightning as L

class LitClassifier(L.LightningModule):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(nn.Flatten(), nn.Linear(28 * 28, 10))

    def training_step(self, batch, batch_idx):
        x, y = batch
        loss = F.cross_entropy(self.net(x), y)
        self.log("train_loss", loss)   # handled + reduced across devices for you
        return loss

    def configure_optimizers(self):
        return torch.optim.Adam(self.parameters(), lr=1e-3)

trainer = L.Trainer(max_epochs=3, accelerator="auto", devices="auto")
trainer.fit(model=LitClassifier(), train_dataloaders=train_loader)
```

Scaling out is a `Trainer` argument change, not a code change: `L.Trainer(accelerator="gpu", devices=8, num_nodes=4, strategy="ddp")`.

## Architecture / How It Works

The repo ships two distinct libraries under one umbrella:

1. **`lightning.pytorch`** (a.k.a. PyTorch Lightning) — the opinionated `LightningModule` + `Trainer` API described above.
2. **`lightning.fabric`** — a much thinner layer. You keep your own hand-written training loop and Fabric only wraps model/optimizer/dataloader for device placement, precision, and distributed strategy (`fabric.setup(...)`, `fabric.backward(loss)`). This is the intended path for foundation-model / LLM / RL loops where the full Trainer is too rigid[^1].

Inside the Trainer, execution is a fixed sequence of **hooks** and **callbacks**. `LightningModule` defines dozens of overridable hooks (`on_train_epoch_start`, `on_before_optimizer_step`, `on_validation_end`, …); `Callback` objects (EarlyStopping, ModelCheckpoint, LearningRateMonitor) subscribe to those same points without touching the module. Understanding the exact order hooks fire in is the main thing separating "Lightning is magic" from "Lightning is debuggable."

Distributed execution is factored into **Strategy** objects: `DDPStrategy`, `FSDPStrategy`, `DeepSpeedStrategy`, plus single-device and `DataParallel`. **Accelerator** objects abstract CPU / CUDA / MPS (Apple Silicon) / TPU. **Precision plugins** handle 16-bit mixed, bf16, and (via DeepSpeed/FSDP) sharded precision. Because these are pluggable, `devices=8, strategy="fsdp"` reconfigures the whole execution model without the `LightningModule` knowing. `self.log(...)` is routed through the active strategy so metric reduction across ranks is automatic (with caveats — see Production Notes).

The historical wrinkle: the project has carried two PyPI distributions (`pytorch-lightning` and `lightning`) with two import roots (`pytorch_lightning` and `lightning.pytorch`) that are the same code. This dual identity, a legacy of the 2.0-era reorg, is the most common source of confusion when reading examples from different years.

## Production Notes

**The two-package / two-import trap.** `import pytorch_lightning as pl` and `import lightning.pytorch as pl` are supposed to be equivalent, but installing both packages, or saving a checkpoint under one import root and loading under the other, can surface class-path mismatches and hyperparameter-restore failures. Pick one package per project and pin it.

**Breaking changes across minors.** The 1.x line changed hook signatures and defaults fairly often; code and StackOverflow answers from 2020–2022 frequently do not run as-is. 2.0 (2023) removed a large amount of deprecated surface and stabilized the API[^4], but upgrades still require reading the changelog rather than assuming semver-minor safety. Checkpoints are not guaranteed loadable across major versions.

**Distributed logging correctness.** `self.log(...)` reduces across ranks only when you ask correctly. For validation metrics under DDP you generally need `sync_dist=True` (or a `torchmetrics` metric object, which handles cross-rank reduction internally); forgetting it silently reports rank-0-only numbers. `on_step` vs `on_epoch` reduction also changes what the logged scalar means.

**DDP process model.** Lightning launches DDP by re-spawning the whole script per device, so top-level side effects run once per process. Data downloads and one-time setup belong in `prepare_data()` (rank-zero) vs `setup()` (every rank). `num_workers` in your DataLoader interacts with the spawn method; misconfiguration produces the classic "dataloader workers exited unexpectedly" hangs.

**Hidden loop, harder debugging.** When training misbehaves, the stack trace goes through Trainer internals, not your code. Budget time to learn hook order, or drop to `lightning.fabric` where the loop is yours. Overriding `training_step` return semantics (returning a dict vs a scalar vs calling `manual_backward`) changes behavior in non-obvious ways; `automatic_optimization = False` is required for multi-optimizer / GAN / RL cases.

**Overhead.** The maintainers quote roughly 300 ms per epoch over raw PyTorch[^1] — negligible for real workloads, but measurable on toy loops, so don't benchmark the framework on MNIST-scale runs.

## When to Use / When Not

**Use when:**
- You have a standard supervised / self-supervised training shape and want checkpointing, mixed precision, and multi-GPU without writing that plumbing.
- You expect to scale a model from single-GPU to multi-node and want that to be a config change (DDP/FSDP/DeepSpeed) rather than a rewrite.
- You want experiment-logger integrations (TensorBoard, W&B, MLflow, Comet) and callbacks (EarlyStopping, LR monitoring) for free.
- Your team benefits from a consistent, reviewable structure across many models.

**Avoid when:**
- Your loop is inherently custom (multi-agent RL, exotic multi-optimizer schedules) — the Trainer will fight you; use `lightning.fabric` or plain PyTorch.
- You want zero abstraction and full visibility into every line of the loop for teaching or debugging.
- You're already committed to the Hugging Face `Trainer` or `accelerate` stack for a transformers-centric workflow.
- The dependency footprint and version-pinning discipline outweigh the boilerplate you'd save on a small script.

## Alternatives

- huggingface/accelerate — use instead when you want to keep your own training loop and only need thin device/precision/distributed wrapping.
- huggingface/transformers — use instead when your work is transformer-centric and the built-in `Trainer` + Hub ecosystem already covers you.
- mosaicml/composer — use instead when speed-up algorithms (curriculum, selective backprop) and throughput are the priority over API structure.
- skorch-dev/skorch — use instead when you want a scikit-learn-compatible `.fit()` API over PyTorch for smaller/classical workflows.
- keras-team/keras — use instead when you want a higher-level, multi-backend API and can leave PyTorch-specific control behind.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-03 | First public release; `LightningModule` + `Trainer` core[^2]. |
| 1.0 | 2020-10 | First stable API commitment[^3]. |
| 1.5 | 2021-11 | Fault-tolerant training (experimental), LightningLite precursor to Fabric. |
| 2.0 | 2023-03 | Major cleanup: deprecated-API removal, `lightning.pytorch` import root, Fabric formalized[^4]. |
| 2.x | 2023–2026 | Ongoing FSDP/DeepSpeed strategy work, precision updates, continuous releases. |

## References

[^1]: PyTorch Lightning README, "Why PyTorch Lightning?" and Fabric section. https://github.com/Lightning-AI/pytorch-lightning
[^2]: Repository creation and early history, Lightning-AI/pytorch-lightning (created 2019-03-31). https://github.com/Lightning-AI/pytorch-lightning
[^3]: William Falcon, "PyTorch Lightning 1.0" — 2020-10. https://lightning.ai/pages/blog/
[^4]: PyTorch Lightning 2.0.0 release notes. https://github.com/Lightning-AI/pytorch-lightning/releases

## Tags

python, pytorch, deep-learning, machine-learning, model-training, distributed-training, gpu, mlops, ai, framework
