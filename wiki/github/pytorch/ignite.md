# pytorch/ignite

> A high-level training loop library for PyTorch that stays a library — you keep the loop, it gives you events, metrics, and distributed plumbing.

[GitHub repo](https://github.com/pytorch/ignite) ·
[Official website](https://pytorch-ignite.ai) ·
[License: BSD-3-Clause](https://github.com/pytorch/ignite/blob/master/LICENSE)

## Overview

PyTorch-Ignite is a high-level helper library for training and evaluating neural networks in PyTorch[^1]. It sits in the same layer as Keras, PyTorch Lightning, fastai, and Catalyst, but makes a deliberate architectural choice that distinguishes it from all of them: it is a *library*, not a *framework*. There is no inversion of control. You write your own `train_step` function — the forward pass, loss, and `optimizer.step()` are yours — and hand it to an `Engine` that runs the loop and fires events at defined points. Nothing is hidden inside a base class you must subclass and override.

The core abstraction is the **Engine**: an object that loops over a data iterable, calls your step function once per batch, and emits `Events` (`STARTED`, `EPOCH_STARTED`, `ITERATION_COMPLETED`, `EPOCH_COMPLETED`, `COMPLETED`, `EXCEPTION_RAISED`, and more) that arbitrary **handlers** subscribe to. A handler is any callable — a lambda, a function, a bound method — so metrics, checkpointing, early stopping, LR scheduling, and experiment logging are all just handlers attached to events. This event/handler model is the project's defining design and its main selling point over callback-inheritance frameworks.

The tradeoff is explicit. Ignite gives you more control and less magic than Lightning at the cost of more boilerplate: there is no `Trainer.fit()` that wires everything for you, and multi-GPU, AMP, and gradient accumulation are things you assemble from provided pieces rather than toggle with a flag. The project is PyTorch-ecosystem official (housed under the `pytorch` org) and NumFOCUS-affiliated, but it is small and community-maintained rather than a large corporate effort.

## Getting Started

```bash
pip install pytorch-ignite
# or: conda install ignite -c pytorch
```

```python
from ignite.engine import Engine, Events
from ignite.metrics import Accuracy, Loss
from ignite.handlers import ModelCheckpoint

# You own the training step — Ignite does not hide it.
def train_step(engine, batch):
    model.train()
    optimizer.zero_grad()
    x, y = batch
    y_pred = model(x)
    loss = criterion(y_pred, y)
    loss.backward()
    optimizer.step()
    return loss.item()

trainer = Engine(train_step)

# A separate engine for evaluation, with metrics attached.
def eval_step(engine, batch):
    model.eval()
    x, y = batch
    with torch.no_grad():
        return model(x), y

evaluator = Engine(eval_step)
Accuracy().attach(evaluator, "accuracy")
Loss(criterion).attach(evaluator, "loss")

@trainer.on(Events.EPOCH_COMPLETED)
def run_validation(engine):
    evaluator.run(val_loader)
    print(engine.state.epoch, evaluator.state.metrics)

# Checkpoint the best model by validation accuracy.
ckpt = ModelCheckpoint("/tmp/models", "best", n_saved=2, require_empty=False)
evaluator.add_event_handler(Events.COMPLETED, ckpt, {"model": model})

trainer.run(train_loader, max_epochs=100)
```

## Architecture / How It Works

**Engine and State.** `Engine.run(data, max_epochs)` iterates the data loader, incrementing `engine.state.iteration` and `engine.state.epoch`. `engine.state` is a mutable bag holding `output` (whatever your step returned), `metrics`, `batch`, `dataloader`, timing info, and any custom fields you set. Handlers read and mutate this state; it is the shared communication channel between the loop and everything attached to it.

**Events and handlers.** Handlers are registered with `engine.add_event_handler(event, fn, *args)` or the `@engine.on(event)` decorator. Events support *filtering* — `Events.EPOCH_COMPLETED(every=5)`, `Events.ITERATION_STARTED(once=100)`, or a custom `event_filter` predicate — and can be OR-combined (`Events.COMPLETED | Events.EPOCH_COMPLETED(every=10)`). You can also register entirely custom events (`EventEnum` + `register_events`) and `fire_event` them from inside your step, e.g. `BACKWARD_COMPLETED`, to hook logic at sub-iteration granularity.

**Metrics.** Metrics follow a `reset` / `update` / `compute` contract and attach to an engine via `metric.attach(engine, name)`. On attach they wire themselves to `EPOCH_STARTED` (reset), `ITERATION_COMPLETED` (update), and `EPOCH_COMPLETED` (compute → write into `state.metrics`). Metrics support arithmetic composition (`(precision * recall * 2 / (precision + recall))`) so an F1 can be built from Precision and Recall without a bespoke class. In distributed runs, metrics `all_reduce`/`all_gather` internally so the reported value is global.

**Handlers library.** Ships `ModelCheckpoint` / `Checkpoint` + `DiskSaver`, `EarlyStopping`, `TerminateOnNan`, `Timer`, `ProgressBar` (tqdm), the parameter schedulers (`PiecewiseLinear`, `CosineAnnealingScheduler`, `create_lr_scheduler_with_warmup`), and `RunningAverage`. Experiment loggers for TensorBoard, MLflow, ClearML, Weights & Biases, Neptune, Polyaxon, and Visdom live under `ignite.handlers` (historically `ignite.contrib.handlers`).

**Distributed (`ignite.distributed`, aliased `idist`).** A unified wrapper over three backends — native `torch.distributed` (nccl/gloo/mpi), Horovod, and PyTorch/XLA (TPU) — behind one API: `idist.auto_model`, `idist.auto_optim`, `idist.auto_dataloader` adapt objects to the active backend, and `idist.Parallel` / `idist.spawn` launch the processes. The same script runs single-GPU, multi-GPU, multi-node, or TPU by changing the launch config, not the code.

## Production Notes

**Attach metrics to the evaluator, not the trainer.** A metric attached to the training engine averages over the whole epoch, including the early high-loss batches, so training "accuracy" reported this way lags reality. For live training metrics use `RunningAverage`; for true evaluation numbers run a separate evaluator engine on held-out data.

**The `contrib` migration is a real upgrade cost.** For years, loggers, schedulers, and several handlers lived under `ignite.contrib`. The 0.5 line moved most of this into `ignite.handlers` and deprecated the old paths[^2]. Code and tutorials written against `from ignite.contrib.handlers import ...` will emit deprecation warnings or break depending on version. Check the import paths for the exact release you pin.

**Checkpoint/resume is object-based and explicit.** `Checkpoint` saves a `to_save` dict of anything exposing `state_dict()`/`load_state_dict()` — model, optimizer, LR scheduler, and the trainer engine itself (so epoch/iteration counters resume correctly). Forgetting to include the engine or the scheduler in `to_save` is the usual cause of "resumed run restarts from epoch 0" or a mis-stepped LR after restart.

**AMP, gradient accumulation, and custom losses are your code.** `create_supervised_trainer` accepts an `amp_mode="amp"` argument for the common path, but anything beyond the default supervised step — multiple optimizers, GANs, accumulation, TBPTT — means writing the `train_step` yourself. This is the library-not-framework tradeoff in practice: no hidden behavior, but no free lunch either.

**Determinism.** Ignite provides `DeterministicEngine` and `ignite.utils.manual_seed`, but full reproducibility still requires the usual PyTorch cuDNN/seed discipline; the engine helps with data-loader seeding across resumes, not with nondeterministic CUDA kernels.

**Maintenance cadence.** The project is actively maintained but by a small core team, and release cadence is slow compared to Lightning — long stretches on the 0.4.x line. Treat it as stable-and-conservative rather than fast-moving; new PyTorch features may take time to get first-class helpers.

## When to Use / When Not

**Use when:**
- You want to keep full, visible control of your training loop and add structure (events, metrics, checkpointing) without ceding it to a framework.
- Your training is non-standard (multiple models/optimizers, GANs, RL, TBPTT) where inversion-of-control frameworks fight you.
- You want one script that scales from CPU to multi-node to TPU via `idist` without rewriting the loop.
- You're already in the PyTorch ecosystem and want a NumFOCUS-affiliated, dependency-light helper.

**Avoid when:**
- You want maximum out-of-the-box automation (`Trainer.fit()`, auto multi-GPU, auto AMP, built-in logging) with minimal wiring — Lightning is a better fit.
- You want a scikit-learn `fit`/`predict` API and pipeline integration — use skorch.
- You want strong opinionated training defaults and tricks baked in — fastai.
- Your team values a large maintainer base and rapid feature turnaround.

## Alternatives

- Lightning-AI/pytorch-lightning — framework with inversion of control and a `Trainer`; use when you want batteries-included automation over manual loop control.
- huggingface/accelerate — thin device/distributed wrapper only; use when you want to keep your entire raw loop and just abstract multi-GPU/TPU placement.
- skorch-dev/skorch — scikit-learn-compatible wrapper; use when you need `fit`/`predict` and sklearn pipeline/grid-search integration.
- fastai/fastai — high-level with opinionated defaults and training tricks; use when you want strong results fast and accept the opinions.
- catalyst-team/catalyst — config-driven DL framework in the same layer; use if you prefer declarative configs, but note reduced activity.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-01 | First public release. Engine/Events/Metrics core[^1]. |
| 0.2.0 | 2019 | Metrics arithmetic, more handlers, `contrib` module. |
| 0.3.0 | 2020-04 | Expanded metrics, experiment loggers under `contrib`. |
| 0.4.0 | 2020-07 | `ignite.distributed` (idist) unified backend API[^3]. |
| 0.4.x | 2020–2024 | Long-lived stable line; incremental metrics/handlers. |
| 0.5.x | 2024– | `contrib` contents merged into core `ignite.handlers`[^2]. |

## References

[^1]: PyTorch-Ignite documentation and project overview. https://pytorch.org/ignite/
[^2]: PyTorch-Ignite `ignite.handlers` reference (former `contrib` handlers/loggers now under core). https://pytorch.org/ignite/handlers.html
[^3]: PyTorch-Ignite distributed helpers (`ignite.distributed`). https://pytorch.org/ignite/distributed.html

## Tags

python, pytorch, deep-learning, machine-learning, neural-networks, training-loop, metrics, distributed-training, model-training, library
