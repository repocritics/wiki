# elixir-nx/axon

> Nx-powered neural networks for Elixir — a Keras/Flax-style layer graph and training loop that compiles to XLA, for teams who want ML inside the BEAM.

[GitHub repo](https://github.com/elixir-nx/axon) ·
[Documentation](https://hexdocs.pm/axon) ·
[License: Apache-2.0](https://github.com/elixir-nx/axon/blob/main/README.md#license)

## Overview

Axon is the neural-network library of the Elixir Nx ecosystem, created by Sean Moriarity and announced in April 2021[^1], a few months after José Valim unveiled Nx itself. It layers three decoupled APIs: a functional core of numerical definitions (`Axon.Layers`, `Axon.Losses`, `Axon.Activations`, `Axon.Initializers`, `Axon.Metrics`), a model-creation API where a model is a plain Elixir struct describing a layer graph, and a training API (`Axon.Loop`) modeled on PyTorch Ignite's event/handler design[^2]. Since v0.6, optimizers live in a separate package, Polaris[^3]. Because every Axon function is an Nx `defn`, models are backend-agnostic: the same graph runs on the pure-Elixir binary backend, JIT-compiles through EXLA to XLA (CPU/GPU/TPU), or runs via Torchx/LibTorch. Axon is also the foundation of Bumblebee, which loads pretrained Hugging Face checkpoints into Axon models[^4].

The defining tradeoff is ecosystem size versus runtime integration. At ~1,700 stars and a small maintainer group, Axon is a niche library next to PyTorch — fewer ops, fewer pretrained models, fewer answered questions. What you get in exchange is ML inside Elixir's supervision trees and concurrency model: batched inference through `Nx.Serving` in the same VM as your Phoenix app, no Python sidecar. Development is steady but unhurried — roughly one minor release per year lately, with v0.8.1 in March 2026 and commits still landing as of July 2026[^5].

## Getting Started

Add Axon plus an Nx compiler to `mix.exs` (the compiler is effectively mandatory for real workloads — see Production Notes):

```elixir
def deps do
  [
    {:axon, "~> 0.8"},
    {:exla, ">= 0.0.0"}   # let mix resolve an EXLA compatible with your Nx
  ]
end
```

Define, build, and train a model:

```elixir
model =
  Axon.input("input", shape: {nil, 784})
  |> Axon.dense(128, activation: :relu)
  |> Axon.dropout(rate: 0.5)
  |> Axon.dense(10, activation: :softmax)

# build → {init_fn, predict_fn}; params initialized from a template
{init_fn, predict_fn} = Axon.build(model, compiler: EXLA)
params = init_fn.(Nx.template({1, 784}, :f32), %{})

# or train with the loop API
trained_state =
  model
  |> Axon.Loop.trainer(:categorical_cross_entropy, Polaris.Optimizers.adamw(learning_rate: 0.005))
  |> Axon.Loop.metric(:accuracy)
  |> Axon.Loop.run(data, %{}, epochs: 10, compiler: EXLA)
```

## Architecture / How It Works

An Axon model is a data structure, not a process or a compiled artifact. Each layer call (`Axon.dense/3`, `Axon.conv/3`, ...) appends a node to a graph held in an `%Axon{}` struct. Nothing executes until `Axon.build/2` lowers the graph into two functions — `init_fn` and `predict_fn` — as Nx numerical definitions. Shape inference was made lazy in v0.2: `init_fn` takes an input template (`Nx.template/2`) instead of the graph eagerly computing shapes, which is what makes complex and multi-input models tractable[^2].

The compiler passed to `build` (or `Axon.Loop.run`) decides where the model runs: EXLA JIT-compiles the whole forward/backward pass to XLA, with gradients coming from Nx's built-in autodiff, not from anything Axon-specific. The architecture is the same as JAX/Flax: pure functions over tensors, traced and compiled as a unit.

`Axon.Loop` structures training as a reducer over data with event hooks (`:iteration_completed`, `:epoch_completed`, ...) for metrics, checkpointing, early stopping, and LR scheduling. Since v0.7, all trainable state travels in an `Axon.ModelState` struct (parameters plus frozen/trainable partitioning) rather than a bare parameter map[^2]. Graph-manipulation APIs (`Axon.rewrite_nodes`, `Axon.pop_nodes`, `Axon.freeze/unfreeze`) support transfer learning and model surgery, and are what Bumblebee leans on to adapt pretrained checkpoints.

## Production Notes

- **EXLA is not optional.** The default binary backend is pure Elixir and orders of magnitude too slow for training or non-toy inference; treat it as a test/CI backend only.
- **JIT cost and shape stability.** First execution compiles through XLA; every new input shape triggers recompilation. Keep batch shapes fixed (pad the last partial batch) or pay repeated compile stalls. `Nx.Serving` handles batching/padding for inference and is the intended production path[^6].
- **Inference is the strong story; large-scale training is not.** Distributed training is still listed as future work in the README[^7]. Single-node, single-accelerator training works; anything bigger is commonly done in Python, with weights imported via Bumblebee, AxonONNX, or Ortex.
- **Serialization moved under your feet.** v0.7 removed `Axon.serialize/deserialize`; the sanctioned pattern is `Nx.serialize` on the model state only, keeping the model definition in code[^2]. Pre-0.7 checkpoints need migration.
- **Upgrade pains are real for a 0.x library.** v0.6 deprecated `Axon.Optimizers`/`Axon.Schedules`/`Axon.Updates` in favor of Polaris and moved input shape to an option; v0.7 replaced parameter maps with `Axon.ModelState`[^2]. Budget for breakage on every minor bump.
- **Channels-last by default** since v0.3 (chosen for performance)[^2] — opposite of PyTorch's NCHW convention; porting conv models requires layout attention at every reshape/permute.
- **Bus factor.** Development is concentrated in a handful of contributors, Sean Moriarity foremost. Issue volume is low (23 open), which reads as a small user base as much as good health.

## When to Use / When Not

**Use when:**
- Your product is already Elixir/Phoenix and you want inference in-process, supervised and batched via `Nx.Serving`, without a Python service.
- You want to fine-tune or run Bumblebee-provided pretrained models with Elixir-native tooling (Livebook notebooks included).
- Your models are modest — classifiers, forecasting, embeddings — and operational simplicity beats ecosystem breadth.

**Avoid when:**
- You need multi-GPU or distributed training; that story does not exist here yet.
- You depend on the long tail of research code, custom CUDA kernels, or day-one paper reimplementations — that is PyTorch territory.
- Your team has no Elixir stake; adopting a BEAM language to get a smaller ML ecosystem is the wrong direction.
- You only need to run an existing ONNX model from Elixir — Ortex does that without pulling in a training framework.

## Alternatives

- elixir-nx/bumblebee — use instead when you want pretrained transformers (LLMs, Whisper, Stable Diffusion) in Elixir rather than building models.
- elixir-nx/ortex — use instead when you just need ONNX Runtime inference from Elixir with no training.
- pytorch/pytorch — use instead when you need the full research ecosystem, distributed training, and maximum pretrained-model coverage.
- keras-team/keras — the closest Python API analog (layer graph + fit loop) if Elixir is not a requirement.
- google/flax — the closest architectural analog (pure functions + XLA JIT) in the JAX world.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-04 | Public announcement, "Axon: Deep Learning in Elixir"[^1]. |
| 0.1.0 | 2022-06-16 | First Hex release[^5]. |
| 0.2.0 | 2022-08-13 | Lazy compilation via `Axon.build`/template-based init[^2]. |
| 0.3.0 | 2022-10-27 | Channels-last default, mermaid graph rendering[^2]. |
| 0.5.0 | 2023-02-16 | Nx bump, removed deprecated `transform`[^2]. |
| 0.6.0 | 2023-08-17 | Optimizers split out to Polaris; input shape as option[^2][^3]. |
| 0.7.0 | 2024-10-08 | `Axon.ModelState`, quantization API, `Axon.serialize` removed[^2]. |
| 0.8.0 | 2025-11-14 | Updated Nx/EXLA/Torchx[^2]. |
| 0.8.1 | 2026-03-11 | `rms_norm` layer, initializer fixes[^2]. |

## References

[^1]: Sean Moriarity, "Axon: Deep Learning in Elixir" — 2021-04-08. https://seanmoriarity.com/2021/04/08/axon-deep-learning-in-elixir/
[^2]: Axon CHANGELOG. https://github.com/elixir-nx/axon/blob/main/CHANGELOG.md
[^3]: Polaris — optimizers extracted from Axon. https://github.com/elixir-nx/polaris
[^4]: Bumblebee — pretrained Axon models from Hugging Face. https://github.com/elixir-nx/bumblebee
[^5]: Axon releases on Hex. https://hex.pm/packages/axon
[^6]: `Nx.Serving` documentation. https://hexdocs.pm/nx/Nx.Serving.html
[^7]: Axon README, "Optimization and training". https://github.com/elixir-nx/axon#readme

## Tags

elixir, deep-learning, neural-networks, nx, xla, machine-learning, training-loop, beam, gpu-acceleration, functional-programming
