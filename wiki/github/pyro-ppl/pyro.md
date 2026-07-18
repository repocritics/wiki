# pyro-ppl/pyro

> Deep universal probabilistic programming on PyTorch — variational inference first, everything else second.

[GitHub repo](https://github.com/pyro-ppl/pyro) ·
[Official website](http://pyro.ai) ·
[License: Apache-2.0](https://github.com/pyro-ppl/pyro/blob/dev/LICENSE.md)

## Overview

Pyro is a probabilistic programming language (PPL) embedded in Python and built on PyTorch. Models are ordinary Python functions containing `pyro.sample` and `pyro.param` statements; inference algorithms manipulate those functions from the outside. It was launched by Uber AI Labs in November 2017[^1], became a Linux Foundation (LF AI) project in February 2019[^2], and is described in a 2019 JMLR paper by Bingham et al.[^3] Today maintenance is community-driven, with a dedicated team at the Broad Institute, where Pyro underpins single-cell genomics tooling.

The design center is **stochastic variational inference (SVI) at deep-learning scale**: minibatch subsampling, reparameterized gradients, and neural networks inside both the model and the approximating "guide". That is also the defining tradeoff — Pyro is the strongest PPL for amortized/variational inference over large data with neural components, but its native HMC/NUTS is slow relative to Stan or its own JAX-based sibling NumPyro, because PyTorch cannot JIT-compile the whole sampling loop the way JAX does. The team's MCMC energy visibly moved to NumPyro after 2019.

At ~9,000 stars it is the most-starred PyTorch-native PPL. The repository is still active (pushed July 2026), but release cadence is slow: the last PyPI release, 1.9.1, dates to June 2024. Treat it as stable and maintained rather than fast-moving.

## Getting Started

```bash
pip install pyro-ppl
```

A minimal model + autoguide + SVI loop (infer the mean of Gaussian data):

```python
import torch, pyro
import pyro.distributions as dist
from pyro.infer import SVI, Trace_ELBO
from pyro.infer.autoguide import AutoNormal
from pyro.optim import Adam

def model(data):
    loc = pyro.sample("loc", dist.Normal(0.0, 10.0))
    with pyro.plate("data", len(data)):
        pyro.sample("obs", dist.Normal(loc, 1.0), obs=data)

data = torch.randn(100) + 3.0
guide = AutoNormal(model)
svi = SVI(model, guide, Adam({"lr": 0.02}), loss=Trace_ELBO())
for step in range(1000):
    svi.step(data)

print(guide.median())  # posterior median for "loc", near 3.0
```

## Architecture / How It Works

The core abstraction is the **effect handler** (the `pyro.poutine` module)[^4]. `pyro.sample` does not draw a random number directly; it emits a message that walks a stack of `Messenger` handlers. Handlers implement everything else: `trace` records executions, `replay` re-runs a model against recorded values, `condition` clamps sites to data, `block` hides sites, `enum` enumerates discrete sites. Inference algorithms are compositions of handlers over plain model functions — this is why the core stays small and why custom inference is possible without forking the library.

On top of that sit the main layers:

- **SVI + ELBO variants** — `Trace_ELBO` (reparameterized gradients), `TraceGraph_ELBO` (score-function estimators with variance reduction for non-reparameterizable sites), `TraceEnum_ELBO` (exact marginalization of discrete latents). The guide is itself a Pyro program; `pyro.infer.autoguide` generates common families (`AutoNormal`, `AutoMultivariateNormal`, etc.) automatically from the model.
- **Discrete enumeration** — since 0.3 (December 2018), Pyro exactly sums out discrete latent variables in parallel, using message passing when the dependency structure has narrow treewidth. This enables Baum-Welch-style HMM training inside a gradient-based pipeline, and is genuinely rare among PPLs[^5].
- **`pyro.plate`** — declares conditional independence, which drives both vectorization and minibatch subsampling with correctly scaled gradients.
- **MCMC** — `HMC`/`NUTS` over PyTorch tensors; correct, but see below.
- **Global parameter store** — `pyro.param` writes into a process-global registry keyed by string name; optimizers mutate it in place.
- **`pyro.contrib`** — GPs, forecasting, epidemiology, Funsor integration; explicitly exempt from the stability guarantee[^6].

## Production Notes

**Shape errors are the dominant footgun.** The `batch_shape` / `event_shape` distinction, `.to_event()`, and plate dimension alignment produce errors (or worse, silently wrong broadcasting) that dominate the learning curve. Read the tensor-shapes tutorial before writing any non-trivial model[^7], and keep `pyro.enable_validation(True)` on during development.

**SVI fails silently.** A converged ELBO does not mean a converged posterior — mean-field autoguides underestimate variance by construction, and bad initialization finds bad local optima. Standard practice: compare `AutoNormal` against `AutoMultivariateNormal` or normalizing-flow guides, vary seeds, and sanity-check against NUTS on a subsampled problem.

**Do not use Pyro's MCMC for large problems.** NUTS in Pyro runs at Python/PyTorch-eager speed per leapfrog step. NumPyro's NUTS, with the identical modeling idiom, is routinely orders of magnitude faster because JAX JIT-compiles the whole kernel[^8]. Teams commonly prototype in one and port to the other; the APIs are close but not identical, so porting is an afternoon, not a rename.

**The global param store bites in notebooks.** Re-running a training cell without `pyro.clear_param_store()` silently reuses stale parameters, including ones whose shapes no longer match your edited model.

**Discrete enumeration cost is exponential in dependent sites.** Enumeration over sequentially dependent discrete variables multiplies; it is only cheap when structure allows message passing or when sites sit in independent plates.

**Release cadence and pinning.** Pyro 1.9.0 (February 2024) dropped PyTorch 1 and Python 3.7[^6]; nothing has shipped since 1.9.1 (June 2024). New PyTorch majors may work but are not promptly certified — pin both packages together. The 1.0 stability statement (November 2019) has largely held: documented core APIs have been stable across the 1.x series[^9].

## When to Use / When Not

**Use when:**
- Your model has neural network components and you want Bayesian treatment of some or all of it (deep generative models, VAEs with structured priors, amortized inference).
- You need minibatch/subsampled variational inference over data too large for full-batch MCMC.
- You have discrete latent structure (mixtures, HMMs, CRFs) and want exact enumeration inside a gradient pipeline.
- You are already in the PyTorch ecosystem and want to share modules, optimizers, and GPU tooling with the rest of your stack.

**Avoid when:**
- Your workflow is classical Bayesian statistics with MCMC as the primary tool — Stan, PyMC, or NumPyro give faster, better-diagnosed sampling.
- You want an approachable API for applied statisticians; Pyro's effect-handler and shape semantics assume comfort with both PyTorch and VI theory.
- You need a fast-moving project with frequent releases; cadence has been roughly yearly at best since 2022.
- Your model is small and fully continuous — simpler tools reach a verified posterior with less code and less expertise.

## Alternatives

- pyro-ppl/numpyro — same modeling idiom on JAX; use instead when MCMC speed matters at all.
- pymc-devs/pymc — use instead for applied-statistics workflows wanting a friendlier API and mature diagnostics.
- stan-dev/stan — use instead when gold-standard HMC and a decade of MCMC diagnostics culture outweigh Python-native integration.
- tensorflow/probability — use instead if your stack is TensorFlow/Keras rather than PyTorch.
- blackjax-devs/blackjax — use instead when you only need samplers over your own JAX log-density, not a modeling language.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2017-11-03 | Initial public release from Uber AI Labs[^1]. |
| 0.2.0 | 2018-04-25 | PyTorch 0.4; `torch.distributions` becomes the distribution backend. |
| 0.3.0 | 2018-12-07 | Parallel/dependent discrete enumeration; vectorized ELBO particles[^5]. |
| 1.0.0 | 2019-11-16 | API stabilization with an explicit cross-version stability statement[^9]. |
| 1.8.0 | 2021-12-14 | `pyro.render_model()`, effect-based autoguides (`AutoNormalMessenger`). |
| 1.9.0 | 2024-02-19 | Drops PyTorch 1 and Python 3.7; partial type hints[^6]. |
| 1.9.1 | 2024-06-02 | Latest release as of mid-2026. |

## References

[^1]: Uber Engineering, "Pyro: Uber AI Labs' Open Source Probabilistic Programming Language" — 2017-11-03. https://eng.uber.com/pyro/
[^2]: Linux Foundation press release, "Pyro Probabilistic Programming Language Becomes Newest LF Deep Learning Project" — 2019-02. https://www.linuxfoundation.org/press-release/2019/02/pyro-probabilistic-programming-language-becomes-newest-lf-deep-learning-project/
[^3]: Bingham et al., "Pyro: Deep Universal Probabilistic Programming", JMLR 20 (2019). http://jmlr.org/papers/v20/18-403.html
[^4]: Pyro docs, "Poutine: A Guide to Programming with Effect Handlers in Pyro". https://docs.pyro.ai/en/stable/poutine.html
[^5]: Pyro 0.3.0 release notes — 2018-12-07. https://github.com/pyro-ppl/pyro/releases/tag/0.3.0
[^6]: Pyro 1.9.0 release notes — 2024-02-19. https://github.com/pyro-ppl/pyro/releases/tag/1.9.0
[^7]: Pyro tutorial, "Tensor shapes in Pyro". https://pyro.ai/examples/tensor_shapes.html
[^8]: NumPyro — Pyro's JAX backend sibling. https://github.com/pyro-ppl/numpyro
[^9]: Pyro 1.0.0 release notes — 2019-11-16. https://github.com/pyro-ppl/pyro/releases/tag/1.0.0

## Tags

python, pytorch, probabilistic-programming, bayesian-inference, variational-inference, mcmc, deep-learning, machine-learning, statistics, ml-library
