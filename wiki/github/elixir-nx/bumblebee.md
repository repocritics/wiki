# elixir-nx/bumblebee

> Pre-trained neural network models for Elixir, loaded directly from the Hugging Face Hub and run on the Nx/Axon stack.

[GitHub repo](https://github.com/elixir-nx/bumblebee) ·
[Docs](https://hexdocs.pm/bumblebee) ·
[License: Apache-2.0](https://github.com/elixir-nx/bumblebee/blob/main/LICENSE)

## Overview

Bumblebee is the Elixir counterpart to Python's `huggingface/transformers`: it
reimplements popular model architectures (BERT, GPT-2, Llama, Whisper, Stable
Diffusion, and others) on top of Axon, the neural-network library in the Nx
numerical-computing stack, and can load the trained parameters for those
architectures straight from the Hugging Face Hub[^1]. It is built and maintained
by Dashbit, José Valim's company, and shipped its first release alongside a
December 2022 announcement covering GPT-2 and Stable Diffusion in Elixir[^2].

The central design fact is that a Hub repository in the Transformers format
stores only a `config.json` and serialized weights (`.safetensors` or PyTorch
`.bin`) — it does not store the model *code*. The architecture implementation
lives in the library. So Bumblebee can only load a model if that architecture has
been ported to Elixir; loading an arbitrary Hub checkpoint is not guaranteed to
work, and the answer to "does model X work?" is "is X's architecture implemented
in Bumblebee?"[^1]. This is the defining tradeoff versus Python: you get the BEAM
runtime, `Nx.Serving` batching, and Phoenix/LiveView integration, at the cost of
a smaller and lagging catalog of supported architectures.

Bumblebee is aimed at Elixir teams who want to run inference (and, increasingly,
serving) inside an existing BEAM application rather than standing up a separate
Python service. It is an inference-and-loading library, not a training framework —
training lives in Axon.

## Getting Started

Add Bumblebee and EXLA to `mix.exs`. EXLA is nominally optional but effectively
required: it JIT-compiles models through Google's XLA to run on CPU or GPU.

```elixir
def deps do
  [
    {:bumblebee, "~> 0.6.0"},
    {:exla, ">= 0.0.0"}
  ]
end
```

```elixir
# config/config.exs — make EXLA the default Nx backend
import Config
config :nx, default_backend: EXLA.Backend
```

```elixir
# Load an architecture Bumblebee implements, build a serving, run it
{:ok, model_info} = Bumblebee.load_model({:hf, "google-bert/bert-base-uncased"})
{:ok, tokenizer}  = Bumblebee.load_tokenizer({:hf, "google-bert/bert-base-uncased"})

serving = Bumblebee.Text.fill_mask(model_info, tokenizer)
Nx.Serving.run(serving, "The capital of [MASK] is Paris.")
#=> %{predictions: [%{score: 0.928, token: "france"}, ...]}
```

For GPUs you must set `XLA_TARGET` (e.g. `cuda12`) so the correct precompiled XLA
binary is fetched[^3]. In notebooks, `Mix.install/2` with a `config:` block does
the same wiring in a single call. Livebook Smart Cells generate this code from a
dropdown, which is the fastest path to a working demo[^2].

## Architecture / How It Works

The stack is layered: **Nx** provides tensors and a pluggable backend/compiler
interface; **Axon** builds the model graph as a functional definition; **Bumblebee**
supplies the concrete architectures plus loaders that map Hub configs and weights
onto Axon models. Inference is dispatched to a backend — almost always **EXLA**,
which lowers the graph to XLA and JIT-compiles it for the CPU or GPU device[^4].

Loading a model is a two-part act. `load_model/2` fetches `config.json`,
constructs the matching Axon graph from Bumblebee's Elixir implementation, then
fetches the parameter tensors and pairs them by name. The name pairing is done by
each architecture's `params_mapping`, which must line up with the Python layer
names in the checkpoint — a mismatch loads garbage or fails, which is why
contributors use `log_params_diff: true` when porting a model[^5].

Tokenization is deliberately outsourced. Bumblebee binds to Hugging Face's Rust
`tokenizers` crate and therefore requires a `tokenizer.json` ("fast tokenizer")
file; it does not implement the Python "slow tokenizer" formats. Repositories
that ship only `tokenizer_config.json` must be converted first, or the tokenizer
borrowed from the base checkpoint[^1].

Deployment is built on **`Nx.Serving`**, which wraps a model into a process that
batches concurrent requests, distributes across GPUs, and can run as a node in a
cluster. This is the piece that makes "embed inference in a Phoenix app" credible:
`Nx.Serving` handles the batching and backpressure that a naive `run/2` loop would
not.

## Production Notes

- **First-call compilation cost.** EXLA JIT-compiles on the first run for a given
  input shape. Cold inference includes that compile; to avoid recompiling per
  request, set `compile:` (batch size and sequence length) and
  `defn_options: [compiler: EXLA]` on the serving so shapes are fixed and the
  computation is compiled once up front. Variable-length inputs that change shape
  every call defeat the cache and recompile repeatedly.
- **The architecture catalog is the real constraint.** Whether a model runs is
  decided by Bumblebee's implemented architectures, not by the Hub. New model
  families appear in Python `transformers` first and are ported later, if at all.
  Verify support before committing — call `load_model/2` against the repo, or use
  the community repository-inspector tool referenced in the docs[^1].
- **Weight format support is partial.** Safetensors and PyTorch `.bin` load;
  Flax (`flax_model.msgpack`) and TensorFlow (`tf_model.h5`) do not[^1]. Some Hub
  repos ship only the latter.
- **Memory and download footprint.** Parameters are downloaded and held; large
  models (Stable Diffusion, multi-billion-parameter LLMs) need corresponding host
  or GPU RAM. There is no built-in quantization comparable to GGUF/llama.cpp, so
  memory use tracks full-precision (or fp16) weights.
- **GPU setup is an XLA concern, not a Bumblebee one.** Driver/CUDA mismatches
  surface as XLA errors at compile time; the `XLA_TARGET` value must match the
  installed CUDA toolkit[^3].
- **Version churn.** The library is pre-1.0 (currently 0.6.x). Minor releases have
  carried API and behavior changes; pin the version and read the CHANGELOG before
  upgrading.

## When to Use / When Not

**Use when:**
- Your application already runs on the BEAM and you want inference in-process
  rather than a separate Python microservice.
- The model architecture you need is on Bumblebee's supported list.
- You want `Nx.Serving` batching, GPU distribution, and Phoenix/LiveView
  integration out of the box.
- You are prototyping in Livebook and want Smart Cells to generate the code.

**Avoid when:**
- You need a model whose architecture is not yet implemented — Python
  `transformers` will have it and Bumblebee may not.
- You need training or fine-tuning of large models — this is an inference/loading
  library; training is Axon's domain and the ecosystem is less mature than PyTorch.
- You depend on quantized (GGUF) inference or the widest possible model coverage —
  llama.cpp / Ollama serve that better.
- Your team has no Elixir footprint; adopting the BEAM only to run a model is
  rarely worth it over a Python service.

## Alternatives

- huggingface/transformers — the Python reference; use it when you need the full,
  up-to-date architecture catalog and training, and Elixir is not a requirement.
- elixir-nx/axon — the underlying Elixir NN library; use it directly when you are
  building or training a model rather than loading a pretrained checkpoint.
- huggingface/candle — Rust-native ML for embedded inference; use when you want a
  compiled binary without a Python or BEAM runtime.
- ggml-org/llama.cpp — use for quantized (GGUF) LLM inference tuned for CPU and
  constrained memory.
- ollama/ollama — use when you want simple local LLM serving over an HTTP API and
  don't need to embed the model in your own process.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2022-12 | First release; announced with GPT-2 and Stable Diffusion in Elixir[^2]. |
| 0.x | 2023–2025 | Iterative additions of architectures (Whisper, Llama, and others), Nx.Serving-based servings, and Hub loading improvements. |
| 0.6.0 | current | Latest release line; installed via `~> 0.6.0`[^1]. |

## References

[^1]: Bumblebee README — model support, tokenizer support, weight formats, installation. https://github.com/elixir-nx/bumblebee
[^2]: Livebook/Dashbit announcement, "Bumblebee: GPT2, Stable Diffusion, and more in Elixir" — 2022-12. https://news.livebook.dev/announcing-bumblebee-gpt2-stable-diffusion-and-more-in-elixir-3Op73O
[^3]: elixir-nx/xla — `XLA_TARGET` usage for CPU/GPU builds. https://github.com/elixir-nx/xla#usage
[^4]: EXLA — Nx backend/compiler targeting Google XLA. https://github.com/elixir-nx/nx/tree/main/exla
[^5]: Bumblebee contributing guide — `params_mapping`, `log_params_diff`, reference-value tests. https://github.com/elixir-nx/bumblebee#contributing

## Tags

elixir, machine-learning, neural-networks, hugging-face, nx, axon, transformers, inference, pretrained-models, beam
