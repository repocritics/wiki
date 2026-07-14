# jax-ml/jax

> Composable function transformations (`grad`, `jit`, `vmap`) over Python+NumPy, compiled to CPU/GPU/TPU via XLA.

[GitHub repo](https://github.com/jax-ml/jax) ·
[Official website](https://docs.jax.dev) ·
[License: Apache-2.0](https://github.com/jax-ml/jax/blob/main/LICENSE)

## Overview

JAX is a Python library for accelerator-oriented array computation and program transformation, built by Google Research (originally Google Brain) and open-sourced in 2018[^1]. Its NumPy-shaped API (`jax.numpy`) is a familiar surface, but the actual product is a set of *composable transformations* of pure Python functions: `jax.grad` (automatic differentiation, forward and reverse mode, to any order), `jax.jit` (trace-and-compile to fused kernels via XLA), and `jax.vmap` (automatic batching). These compose arbitrarily — `jit(vmap(grad(f)))` is idiomatic — which is the whole thesis of the project.

The defining tension is that JAX buys its speed and composability by demanding **functional purity**. Traced functions must have no side effects, arrays are immutable, and randomness is explicit and stateless. Code that would be natural in NumPy or PyTorch (in-place mutation, Python `if` on array values inside `jit`, a global RNG) either silently misbehaves or raises under tracing. The project itself calls these the "sharp bits" and ships a dedicated gotchas notebook[^2]. JAX is a low-level substrate, not a batteries-included deep-learning framework — neural-network layers, optimizers, and checkpointing live in separate ecosystem libraries (Flax, Optax, Orbax).

JAX is the compute backbone for a large share of Google/DeepMind research and TPU-scale training, and has broad external adoption in scientific computing, probabilistic programming, and ML research where the transform-composition model and TPU support matter more than ecosystem breadth. The repository was transferred from `google/jax` to the vendor-neutral `jax-ml` org in 2024; old URLs still redirect.

## Getting Started

```bash
pip install -U jax                 # CPU
pip install -U "jax[cuda13]"       # NVIDIA GPU (CUDA via pip wheels)
pip install -U "jax[tpu]"          # Google TPU
```

```python
import jax
import jax.numpy as jnp

def loss(w, x, y):
    preds = jnp.dot(x, w)
    return jnp.mean((preds - y) ** 2)

# Compose: reverse-mode gradient, then JIT-compile the whole thing.
grad_loss = jax.jit(jax.grad(loss))

w = jnp.zeros(3)
x = jnp.ones((16, 3))
y = jnp.ones(16)
print(grad_loss(w, x, y).shape)   # (3,)
```

Randomness is explicit — there is no global seed:

```python
key = jax.random.key(0)
key, subkey = jax.random.split(key)   # never reuse a key
noise = jax.random.normal(subkey, (3,))
```

## Architecture / How It Works

JAX works by **tracing**. When you call a transformed function, JAX runs your Python once with abstract `Tracer` objects standing in for real arrays, recording the primitive operations into an intermediate representation (a `jaxpr`). Transformations are rewrites of that trace:

- **`grad`** applies the chain rule over the jaxpr (reverse-mode builds and replays a linearized backward pass).
- **`jit`** hands the jaxpr to **XLA**, which fuses operations, allocates buffers, and emits a single optimized kernel per (shape, dtype) signature.
- **`vmap`** rewrites primitives to operate over an added batch axis, turning per-example code into batched code without a Python loop.

Because tracing only sees *operations on array values*, Python control flow that branches on a traced value cannot be captured — hence `jax.lax.cond`, `jax.lax.scan`, and `jax.lax.while_loop` as structured replacements inside `jit`. Values known at trace time (shapes, `static_argnums`) are baked in as constants; everything else must flow through as array data.

Two consequences fall out of the design:

- **Immutable arrays.** `x[0] = 1` is unavailable; the functional form is `x = x.at[0].set(1)`, which returns a new array (XLA elides the copy when it can).
- **PyTrees.** Parameters, optimizer state, and batches are arbitrary nested dicts/lists/tuples of arrays. Transformations map over these structures via the pytree registry, which is why JAX code passes plain nested containers around instead of framework objects.

Dispatch is **asynchronous**: calls return immediately with a future-like `jax.Array`, and Python races ahead while the accelerator computes. Blocking happens only when you print, convert to NumPy, or call `.block_until_ready()`.

For multi-device execution, the modern path is a single `jax.Array` sharded across a device `Mesh`, with `jit` performing automatic partitioning (GSPMD/`shard_map` for manual control). This unified array model replaced the older `pmap` + device-array approach; `pmap` still exists but is legacy for most new code.

## Production Notes

**Recompilation is the number-one footgun.** `jit` caches compiled executables keyed on input *shapes and dtypes*. Feeding variable-length batches, changing sequence lengths, or passing Python scalars that vary triggers a fresh (slow) XLA compile every call — sometimes silently dominating runtime. Fixes: pad to fixed shapes, mark truly-static args with `static_argnums`, and watch for retracing with `jax.jit`'s cache or logging.

**Compile latency.** XLA compilation of large models can take seconds to minutes on the first call; this is invisible in microbenchmarks but real in short jobs and test suites. `jax.jit(...).lower(...).compile()` lets you compile ahead of time, and persistent compilation caches (`jax.config.update("jax_compilation_cache_dir", ...)`) amortize it across runs.

**float64 is off by default.** JAX silently coerces to float32 even when you pass float64 inputs, unless you set `jax_enable_x64`. Scientific-computing users are routinely surprised by this. Set it at startup, before any array is created.

**Debugging under `jit` is different.** Ordinary `print` runs at trace time and shows Tracers, not values. Use `jax.debug.print` / `jax.debug.breakpoint` for runtime values, or `jax.disable_jit()` to fall back to eager execution while diagnosing.

**GPU/TPU install is version-sensitive.** The CUDA wheels (`jax[cudaXX]`) pin CUDA/cuDNN combinations; mismatches with a system CUDA or with a co-installed PyTorch CUDA build are a common source of "no GPU found" or memory-allocator conflicts. JAX also preallocates ~75% of GPU memory by default (`XLA_PYTHON_CLIENT_PREALLOCATE=false` / `MEM_FRACTION` to change).

**Pre-1.0, but stable in practice.** JAX is still on 0.x versioning and does ship breaking changes across minor releases (deprecations, sharding API evolution, `jax.Array` unification). Pin versions and read the changelog before upgrading; the churn is real but the core transform API has been stable for years.

**No neural-net layer in core.** Choose an ecosystem library: Flax (`nnx`/`linen`) is the mainstream Google choice, Equinox a lighter pytree-native option, Optax for optimizers, Orbax for checkpointing. Keras 3 also runs on a JAX backend for a higher-level API.

## When to Use / When Not

**Use when:**
- You want composable autodiff + JIT + batching and control over the numerics (research, custom training loops, novel architectures).
- You target TPUs, or need large-scale SPMD parallelism with a global-array programming model.
- You're doing scientific / differentiable-programming work (physics, optimization, probabilistic models) where NumPy semantics + gradients matter.

**Avoid when:**
- You want a batteries-included framework with layers, data loaders, and a training loop out of the box — reach for PyTorch.
- Your workload leans on dynamic shapes, in-place mutation, or heavy Python control flow that fights the tracing model.
- You need the largest model/tutorial/pretrained-weight ecosystem, or the broadest hiring pool — that is still PyTorch.

## Alternatives

- pytorch/pytorch — use instead when you want eager execution, dynamic shapes, and the largest ecosystem of models, tutorials, and pretrained weights.
- tensorflow/tensorflow — use instead when you need mature production serving, mobile/edge deployment (TFLite), and end-to-end pipeline tooling (TFX).
- keras-team/keras — use instead when you want a high-level layer/model API; Keras 3 can even run on a JAX backend.
- tinygrad/tinygrad — use instead when you want a tiny, hackable, readable autodiff+accelerator stack over maximal features.
- google/flax — not an alternative but the standard neural-network layer library built on top of JAX; pair the two.

## History

| Version | Date | Notes |
|---------|------|-------|
| (Autograd) | 2015 | Predecessor by the same authors — NumPy autodiff, no compilation[^1]. |
| initial | 2018-10 | JAX open-sourced; `grad`/`jit`/`vmap` over XLA. MLSys/SysML paper on high-level tracing[^3]. |
| 0.4.0 | 2022-12 | Unified `jax.Array` type; single global sharded-array model, superseding device-array/`pmap`-centric APIs[^4]. |
| (transfer) | 2024 | Repository moved `google/jax` → `jax-ml/jax` (vendor-neutral org); old URLs redirect. |

## References

[^1]: JAX README and project description — "Composable transformations of Python+NumPy programs." https://github.com/jax-ml/jax
[^2]: "Common Gotchas in JAX" — sharp-bits notebook. https://docs.jax.dev/en/latest/notebooks/Common_Gotchas_in_JAX.html
[^3]: Bradbury et al., "JAX: Autograd and XLA / Compiling machine learning programs via high-level tracing" — SysML/MLSys 2018. https://mlsys.org/Conferences/2019/doc/2018/146.pdf
[^4]: JAX changelog. https://docs.jax.dev/en/latest/changelog.html

## Tags

python, machine-learning, autodiff, jit-compilation, xla, gpu, tpu, numerical-computing, deep-learning, functional, accelerator, scientific-computing
