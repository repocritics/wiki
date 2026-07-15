# elixir-nx/nx

> Multi-dimensional arrays (tensors) and JIT-compiled numerical definitions for Elixir — the NumPy/JAX layer the BEAM never had.

[GitHub repo](https://github.com/elixir-nx/nx) ·
[License: Apache-2.0](https://github.com/elixir-nx/nx/blob/main/nx/LICENSE)

## Overview

Nx ("Numerical Elixir") brings tensor computation and automatic differentiation to Elixir. It was started in 2020 by José Valim (creator of Elixir) and Sean Moriarity, and publicly announced in February 2021[^1]. The core idea is `defn` — a numerical definition, a restricted subset of Elixir that Nx traces into a computation graph and hands to a pluggable compiler backend for just-in-time compilation to CPU, GPU, or TPU. In shape this is closer to Google's JAX than to NumPy: the same source runs eagerly for debugging and compiles to fused accelerator kernels in production, and `grad/2` gives reverse-mode autodiff over any `defn`[^2].

Nx is the foundation of an ecosystem rather than an end-user tool. Axon (neural networks), Bumblebee (pretrained Transformer models from Hugging Face), Explorer (dataframes), and Scholar (classical ML) all sit on top of it, and Livebook is the usual interactive surface[^3]. If you are doing machine learning in Elixir, you are using Nx transitively whether or not you write tensor code yourself.

The defining tension is maturity versus ambition. Nx is still pre-1.0 (0.12.1 as of mid-2026) and the public API can and does shift between minor versions. It is competing conceptually with a Python numerical stack that has a fifteen-year head start and vastly more contributors, models, and Stack Overflow answers. What it offers in return is genuinely different: numerical code that inherits the BEAM's concurrency, distribution, fault tolerance, and Phoenix/LiveView integration, without a Python sidecar process.

## Getting Started

Nx is a Hex package. The backend is a separate dependency — the pure-Elixir `BinaryBackend` ships by default, but real work uses EXLA (Google's XLA) or Torchx (LibTorch).

```elixir
# mix.exs
def deps do
  [
    {:nx, "~> 0.12"},
    {:exla, "~> 0.12"}   # XLA-compiled backend: CPU, CUDA, ROCm, TPU
  ]
end
```

```elixir
# Route all tensor ops through XLA instead of the pure-Elixir default
Nx.global_default_backend(EXLA.Backend)

defmodule Math do
  import Nx.Defn

  # A numerical definition — traced and JIT-compiled, not interpreted
  defn softmax(t) do
    Nx.exp(t) / Nx.sum(Nx.exp(t))
  end

  # Reverse-mode automatic differentiation of an arbitrary function
  defn dcube(x), do: grad(x, fn x -> x ** 3 end)
end

Math.softmax(Nx.tensor([1.0, 2.0, 3.0]))
Math.dcube(Nx.tensor(2.0))   # => #Nx.Tensor<f32 12.0>
```

Requires Elixir `~> 1.17`[^4]. EXLA downloads precompiled XLA binaries at build time keyed to your OS/CUDA combination.

## Architecture / How It Works

Nx separates the tensor *API* from the *backend* that executes it. Every `Nx.Tensor` struct carries a reference to a backend; calling `Nx.add/2` dispatches to that backend's implementation. `Nx.BinaryBackend` executes in pure Elixir on the BEAM (correct, portable, slow); EXLA and Torchx hand the work to compiled native libraries and keep the actual buffers off-heap, in device memory[^2].

`defn` is where the leverage lives. Inside a `defn`, ordinary-looking Elixir is not executed — it is *traced*. Operators like `+` and `*` are overridden to build an expression tree (`Nx.Defn.Expr`), which a compiler lowers to its target. Under EXLA that target is an XLA computation, which XLA fuses and optimizes into a single kernel. This is why `defn` can be an order of magnitude faster than the equivalent eager calls: the whole function compiles as one graph rather than many dispatched operations. `Nx.Defn.jit/2` compiles and caches by input shape; `grad/2` and `value_and_grad/2` differentiate the traced expression symbolically.

Because `defn` is a restricted language, not all Elixir works inside it: control flow uses `Nx.Defn` constructs (`if`, `while`, `cond`), data must be tensors, and side effects are disallowed. This is the same bargain JAX makes — the restriction is what makes tracing and compilation tractable.

The repository is a monorepo holding three published packages: `nx` itself, `exla` (the XLA backend), and `torchx` (the LibTorch backend). The README notes these will eventually be extracted into their own repositories[^5].

## Production Notes

**Backend installation is the recurring pain point.** EXLA fetches prebuilt XLA archives at compile time; getting the CUDA, cuDNN, and XLA versions to line up on a specific GPU driver is the most common source of setup failures, and mismatches surface as opaque runtime errors rather than clear build failures. Pin `XLA_TARGET` (`cuda12`, `rocm`, `cpu`, `tpu`) explicitly and expect a slow first build while archives download and compile.

**Pre-1.0 means breaking changes.** Nx has shipped API changes and behavior shifts across minor releases (e.g. 0.4→0.5, 0.6→0.7). Upstream libraries like Axon and Bumblebee pin narrow Nx version ranges, so upgrading Nx often means upgrading the whole stack together rather than in isolation. Read the CHANGELOG before bumping.

**Eager vs. compiled performance gap is large.** Tensor code written outside `defn`, or run on the default `BinaryBackend`, can be thousands of times slower than the same logic JIT-compiled through EXLA. Set a global default backend early and put hot numerical paths inside `defn`; do not benchmark on `BinaryBackend`.

**JIT compilation is cached by shape.** Every distinct input shape triggers a fresh compilation. Highly variable shapes (e.g. ragged batch sizes) cause repeated recompilation stalls; pad or bucket inputs to a fixed set of shapes in serving paths.

**GPU memory is preallocated.** XLA, like TensorFlow, defaults to grabbing a large fraction of GPU memory up front. Multi-tenant GPU hosts need `XLA_FLAGS` / client memory options tuned, or processes will contend.

## When to Use / When Not

**Use when:**
- Your application is already on Elixir/Phoenix and you want ML or numerical code in-process rather than through a Python service.
- You want autodiff and accelerator compilation (`defn` + EXLA) with JAX-like ergonomics on the BEAM.
- You need to serve models (via Bumblebee/Nx.Serving) with the BEAM's concurrency and distribution handling the request layer.

**Avoid when:**
- You need the breadth of the Python ecosystem — arbitrary pretrained models, exotic layers, research code — where PyTorch/JAX are the lingua franca.
- You require a frozen, 1.0-stable numerical API; Nx still moves between minor versions.
- Your team has no Elixir footprint; adopting the BEAM solely to run tensors is rarely worth it over NumPy/PyTorch.

## Alternatives

- google/jax — the direct conceptual model for `defn`: composable `jit` + `grad` over XLA. Use it when you are in Python and want the mature version of the same idea.
- pytorch/pytorch — use when you need the dominant deep-learning ecosystem, eager debugging, and the largest model/library selection.
- numpy/numpy — use when you only need CPU array math in Python and want no compiler or accelerator machinery.
- tensorflow/tensorflow — also XLA-backed like EXLA, but Python-first with a graph/Keras stack; use when you want that ecosystem rather than the BEAM.
- elixir-nx/axon — not a substitute but the layer above Nx; use it when you want neural-network definitions instead of raw tensor ops in Elixir.

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo) | 2020-10-29 | Repository created; project begun by Valim + Moriarity[^1]. |
| announce | 2021-02 | Nx publicly announced[^1]. |
| 0.1.0 | 2022-01-06 | First Hex release. |
| 0.5.0 | 2023-02-10 | API refinements; ongoing pre-1.0 churn. |
| 0.7.0 | 2024-02-22 | Continued backend/defn evolution. |
| 0.9.0 | 2024-09-26 | — |
| 0.11.0 | 2026-02-19 | — |
| 0.12.1 | 2026-05-22 | Latest release as of writing[^6]. |

## References

[^1]: José Valim, "Nx" announcement — February 2021. https://dashbit.co/blog/nx-numerical-elixir-is-now-publicly-available
[^2]: Nx package README and `Nx.Defn` documentation. https://hexdocs.pm/nx/Nx.Defn.html
[^3]: elixir-nx organization overview (Axon, Bumblebee, Explorer, Scholar, Livebook). https://github.com/elixir-nx
[^4]: `nx/mix.exs`, `elixir: "~> 1.17"`, `@version "0.12.1"`. https://github.com/elixir-nx/nx/blob/main/nx/mix.exs
[^5]: Repository root README — monorepo of Nx, EXLA, Torchx, "extracted to their own repository in the future." https://github.com/elixir-nx/nx#readme
[^6]: GitHub releases, v0.12.1 published 2026-05-22. https://github.com/elixir-nx/nx/releases

## Tags

elixir, tensor, machine-learning, numerical-computing, autodiff, jit, gpu, xla, beam, jax-like
