# keras-team/keras

> High-level deep learning API that runs the same model code on JAX, TensorFlow, or PyTorch.

[GitHub repo](https://github.com/keras-team/keras) ·
[Official website](https://keras.io/) ·
[License: Apache-2.0](https://github.com/keras-team/keras/blob/master/LICENSE)

## Overview

Keras is a high-level neural-network API created by François Chollet, first released in March 2015[^1]. It optimizes for readability and iteration speed: you compose layers, call `compile()` and `fit()`, and the framework handles the training loop, autodiff, device placement, and serialization. For a decade it has been the entry point most people use to learn deep learning, and it remains the API that gets a working model to `fit()` in the fewest lines.

The defining fact of Keras today is its history of backend churn. The original 1.x/2.x line was multi-backend (Theano, then TensorFlow, then CNTK). In 2019 it collapsed into TensorFlow: `tf.keras` became the canonical implementation and the standalone package effectively froze[^2]. Then in November 2023 the project reversed course again with **Keras 3**, a ground-up rewrite that is once more multi-backend — this time JAX, TensorFlow, and PyTorch, plus OpenVINO for inference[^3]. The same `keras` package on PyPI now means something very different depending on the year, and a large body of tutorials and Stack Overflow answers refer to the TensorFlow-only era.

The tradeoff Keras asks you to accept is abstraction for control. You get a backend-agnostic, concise modeling layer; you give up direct access to the underlying framework's idioms unless you deliberately drop down to them. This is a good trade for standard architectures and a poor one when you need to do something the abstraction did not anticipate.

## Getting Started

Keras 3 requires a backend framework installed alongside it:

```bash
pip install --upgrade keras
pip install tensorflow   # or: jax  |  torch
```

Select the backend before the first import (it cannot be changed afterward):

```python
import os
os.environ["KERAS_BACKEND"] = "jax"   # "tensorflow" | "torch" | "openvino"

import keras
from keras import layers

model = keras.Sequential([
    layers.Input((784,)),
    layers.Dense(128, activation="relu"),
    layers.Dropout(0.2),
    layers.Dense(10, activation="softmax"),
])

model.compile(optimizer="adam",
              loss="sparse_categorical_crossentropy",
              metrics=["accuracy"])
model.fit(x_train, y_train, epochs=5, validation_split=0.1)
```

Backend selection can also live in `~/.keras/keras.json` or the `KERAS_BACKEND` environment variable.

## Architecture / How It Works

Keras exposes three model-authoring styles: the **Sequential** API (a linear stack), the **Functional** API (a graph built by calling layers on symbolic tensors, which enables shape inference and static validation), and **Model subclassing** (imperative `call()` for arbitrary control flow). All three produce a `keras.Model` with the same `fit`/`evaluate`/`predict`/`save` surface.

The pivot that makes Keras 3 backend-agnostic is **`keras.ops`**: a large reimplementation of the NumPy API surface plus neural-network primitives, which dispatches to whichever backend is active[^3]. Custom layers written against `keras.ops` (instead of `tf.*` or `torch.*`) run unchanged across all three backends. Layers hold weights as `keras.Variable`; the framework tracks them for autodiff and checkpointing regardless of backend.

Training is where the backends leak through by design. `fit()` compiles a backend-appropriate step: an XLA-compiled function under JAX, a `tf.function` graph under TensorFlow, an eager or compiled loop under PyTorch. If you need a custom loop, Keras supports overriding `train_step`, but the override is written in the native backend's style — so a custom `train_step` is *not* automatically portable, unlike layers written in `keras.ops`. This is the seam to understand: the declarative model is portable; hand-written training internals are not.

Keras 3 also ships a `keras.distribution` API for data and model parallelism across GPUs/TPUs. Its multi-device support is strongest on JAX, which is the main reason the project markets JAX as the performance backend[^3]. Pretrained models and tokenizers live in a separate package, **KerasHub** (the merger of the former KerasCV and KerasNLP lines), built on top of the core.

## Production Notes

**The backend is a hard, one-time decision per process.** It must be set before `import keras` and cannot change afterward. Notebooks that re-run the import cell after switching backends silently keep the first one; the fix is a fresh kernel. Library code that imports Keras at module load can override an application's intended backend if it sets `KERAS_BACKEND` itself.

**"Keras" means three different products.** `pip install keras` gives you Keras 3. Legacy TensorFlow-only Keras 2 is now the separate `tf-keras` package, and `tf.keras` inside TensorFlow may resolve to either depending on the TensorFlow version's `TF_USE_LEGACY_KERAS` setting. Reproducing an old result often requires pinning `tf-keras`, not `keras`. Much online documentation predates the split and is silently wrong for Keras 3.

**Saved-model formats changed.** Keras 3 standardizes on the `.keras` zip archive; the older HDF5 (`.h5`) and TensorFlow SavedModel paths still load in many cases but are not the forward path. Models with custom layers require those classes to be importable at load time (or registered via `@keras.saving.register_keras_serializable`), a common deployment footgun.

**Backend behavior is not identical.** The same architecture can differ in numerical precision, default dtypes, RNG seeding, and which ops are supported. A model that trains cleanly on TensorFlow may hit an unimplemented `keras.ops` path or a different mixed-precision default on another backend. Benchmark and validate on the backend you will actually ship, not the one you prototyped on.

**Performance claims are architecture-dependent.** The advertised speedups from choosing "the fastest backend" are real but vary widely by model shape; JAX's advantage comes largely from XLA fusion and shows most on TPU and large batched workloads. Small models and heavy Python-side data pipelines often see no difference.

**Data pipelines stay native.** Keras does not replace `tf.data`, PyTorch `DataLoader`, or `jax` input pipelines — `fit()` consumes any of them. Cross-backend portability of the *model* does not extend to the *input pipeline*, which remains framework-specific in practice.

## When to Use / When Not

**Use when:**
- You want standard architectures (CNNs, transformers, MLPs) trained with minimal boilerplate.
- You want to keep model code portable across JAX/TF/PyTorch and defer the backend choice.
- You are teaching, prototyping, or iterating and value `compile`/`fit` over hand-written loops.
- You need a stable high-level API on top of JAX without adopting Flax's functional style.

**Avoid when:**
- You need low-level control over the training step, memory, or custom kernels — use the backend framework directly.
- Your team is already deep in idiomatic PyTorch; Lightning or bare PyTorch fits the mental model better.
- You depend on the large pretrained-model and fine-tuning ecosystem — Hugging Face Transformers is the center of gravity there.
- You must run legacy `tf.keras` code unchanged; pin `tf-keras` rather than adopting Keras 3.

## Alternatives

- pytorch/pytorch — use directly when you want full control over the training loop and the dominant research ecosystem.
- tensorflow/tensorflow — use its lower-level APIs when you need graph/XLA control beyond what the Keras abstraction exposes.
- google/flax — use for JAX-native research when you prefer explicit functional state over Keras's object model.
- huggingface/transformers — use when your work is loading, fine-tuning, and serving pretrained models rather than authoring architectures.
- Lightning-AI/pytorch-lightning — use when you want structured training loops but want to stay in native PyTorch.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015-03 | Initial release; Theano backend[^1]. |
| 1.0.0 | 2016-04 | API stabilization; TensorFlow backend option matured. |
| 2.0.0 | 2017-03 | TensorFlow becomes the default backend; API cleanup. |
| 2.3.0 | 2019-09 | Last multi-backend release of the 2.x line; users steered to `tf.keras`[^2]. |
| — | 2019 | Folds into TensorFlow 2.0 as `tf.keras`; standalone package effectively frozen. |
| 3.0 | 2023-11 | Multi-backend reboot: JAX, TensorFlow, PyTorch; `keras.ops` layer[^3]. |
| 3.x | 2024–2026 | OpenVINO inference backend, `keras.distribution`, KerasHub consolidation. |

## References

[^1]: François Chollet, Keras project — initial public release, March 2015. https://github.com/keras-team/keras/releases
[^2]: Keras 2.3.0 release notes — final multi-backend 2.x release, recommends `tf.keras`, September 2019. https://github.com/keras-team/keras/releases/tag/2.3.0
[^3]: "Introducing Keras 3.0" — multi-backend architecture over JAX/TensorFlow/PyTorch, November 2023. https://keras.io/keras_3/

## Tags

python, deep-learning, machine-learning, neural-networks, jax, tensorflow, pytorch, keras, high-level-api, model-training
