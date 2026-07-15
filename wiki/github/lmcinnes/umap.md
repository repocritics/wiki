# lmcinnes/umap

> Nonlinear dimension reduction via fuzzy topological structure — the standard t-SNE alternative for embedding and visualization, packaged as a scikit-learn transformer.

[GitHub repo](https://github.com/lmcinnes/umap) ·
[Documentation](https://umap-learn.readthedocs.io) ·
[License: BSD-3-Clause](https://github.com/lmcinnes/umap/blob/master/LICENSE.txt)

## Overview

UMAP (Uniform Manifold Approximation and Projection) is a dimension reduction
technique by Leland McInnes and John Healy of the Tutte Institute, first
published in 2018[^1]. It grew out of a specific dissatisfaction with t-SNE:
UMAP is generally faster, scales to larger datasets, and claims to preserve more
global structure, while exposing a scikit-learn-compatible `fit`/`transform` API
that lets new points be projected into an existing embedding. Installed as the
PyPI package `umap-learn`, it has become the default 2D projection tool in
single-cell genomics, NLP embedding inspection, and general exploratory data
analysis.

The library's defining tension is between how it is marketed and how it is
commonly (mis)used. UMAP is grounded in real manifold-learning theory[^1], but
in practice most users treat the 2D scatter plot as ground truth. It is not:
inter-cluster distances, cluster sizes, and cluster density in a UMAP plot are
largely artifacts of hyperparameters, not properties of the data[^2]. UMAP is
excellent at answering "roughly how many groups are here and which points are
neighbors," and unreliable at answering "how far apart are these groups." The
package itself is honest about this in its docs; the surrounding literature
often is not.

A second tension is the Numba dependency. UMAP is pure Python JIT-compiled with
Numba (it moved off Cython early for clarity and speed), which makes the code
readable and the runtime fast — but adds a multi-second compilation warmup on
first call and couples the project's reliability to Numba/LLVM version support.

## Getting Started

```bash
pip install umap-learn            # core
pip install umap-learn[plot]      # + matplotlib/datashader/holoviews
conda install -c conda-forge umap-learn
```

```python
import umap
from sklearn.datasets import load_digits

digits = load_digits()

# Drop-in for sklearn's t-SNE. Defaults: n_neighbors=15, min_dist=0.1.
reducer = umap.UMAP(n_neighbors=15, min_dist=0.1, metric="euclidean")
embedding = reducer.fit_transform(digits.data)   # (n_samples, 2)

# Project new, unseen points into the SAME embedding space.
more = reducer.transform(digits.data[:10])
```

The two parameters that dominate output are `n_neighbors` (small = local detail,
large = global shape; sensible range 5–50) and `min_dist` (how tightly points
may pack; 0.0–0.5).

## Architecture / How It Works

UMAP runs in two phases. First it builds a weighted k-nearest-neighbor graph in
the original space, using approximate nearest-neighbor search via `pynndescent`
(a sibling project by the same author)[^3]. Local distances are converted into
fuzzy membership strengths — each point gets a locally adaptive notion of
"neighbor," which is what makes the method scale-invariant to varying density.
The per-point fuzzy simplicial sets are then merged into a single global graph.

Second, it optimizes a low-dimensional layout so that its fuzzy topological
structure matches the high-dimensional one, minimizing a cross-entropy between
the two edge-weight distributions via stochastic gradient descent with negative
sampling — mechanically similar to a force-directed graph layout (attraction
along graph edges, repulsion between random pairs). This is why UMAP output
looks like a spring-embedded graph and why its global geometry is emergent
rather than metric.

Everything performance-critical is JIT-compiled with Numba, including
user-supplied distance functions (a custom metric must itself be Numba-jittable).
The core class subclasses sklearn's `BaseEstimator`/`TransformerMixin`, so UMAP
slots into `Pipeline` and `GridSearchCV`. Beyond the base algorithm the package
ships several extensions that reuse the same graph machinery: **supervised /
semi-supervised** reduction (pass labels as `y`); **densMAP** (`densmap=True`),
which adds a density-preservation term so local density in the plot is
meaningful[^4]; **Parametric UMAP**, which trains a TensorFlow/Keras network to
learn the embedding function (enabling fast inference and an inverse transform);
and an experimental inverse transform and non-Euclidean (e.g. hyperbolic) output
spaces.

## Production Notes

The differentiator between a good and a bad UMAP result is almost entirely
operational discipline. Real caveats:

- **The embedding is not a metric space.** Distances between clusters and the
  size/spread of clusters are not interpretable. Do not measure them, and be
  skeptical of any downstream analysis that does[^2].
- **Numba JIT warmup.** The first `fit` in a fresh process pays several seconds
  of compilation before any real work; benchmarks that ignore warmup are
  misleading. In short-lived workers this cost recurs every process.
- **Determinism costs parallelism.** UMAP is stochastic; runs differ. Setting
  `random_state` makes it reproducible but forces single-threaded execution,
  which can be substantially slower on large data. You choose reproducibility or
  speed, not both.
- **`transform()` drifts.** Projecting new points into an existing embedding is
  approximate and can place points oddly if the new data distribution differs
  from training. For a stable, repeatable transform function, prefer Parametric
  UMAP.
- **Hyperparameter sensitivity.** `n_neighbors` and `min_dist` change the
  picture qualitatively. Publish the parameters with any figure; a UMAP plot
  without them is not reproducible.
- **Do not cluster naively on UMAP coordinates.** Clustering the 2D output can
  invent or merge groups. If clustering is the goal, cluster in a
  higher-dimensional UMAP output (e.g. 10–50 components) or with HDBSCAN, and
  treat the 2D plot as visualization only[^5].
- **Heavy optional dependencies.** `[plot]` pulls datashader/holoviews;
  Parametric UMAP pulls TensorFlow. Keep these out of minimal deployment images.
- **Numba/LLVM coupling.** Environment breakage after a NumPy or Numba upgrade
  is the most common install-time issue; pin versions in reproducible pipelines.
- **GPU is out-of-tree.** The core library is CPU-only. For GPU acceleration use
  RAPIDS cuML's UMAP or the TorchDR reimplementation (see Alternatives).

## When to Use / When Not

**Use when:**
- You need to visualize high-dimensional data (embeddings, single-cell, images)
  and want faster, more scalable behavior than t-SNE.
- You need to project new points into an existing embedding, or use reduction as
  a preprocessing step inside an sklearn pipeline.
- You want supervised/semi-supervised reduction using partial labels.

**Avoid when:**
- You need faithful global distances or a metric-preserving projection — use PCA,
  MDS, or a purpose-built method.
- You need strict determinism at full speed (the two conflict here).
- Your only goal is clustering — cluster the data directly rather than its 2D
  projection.
- You want a zero-heavy-dependency install; the Numba/LLVM stack is nontrivial.

## Alternatives

- pavlin-policar/openTSNE — fast, extensible t-SNE with an `transform` for new
  points; use when you specifically want t-SNE semantics or better-understood
  local structure.
- YingfanWang/PaCMAP — dimension reduction tuned to balance local and global
  structure; use when UMAP's global geometry feels untrustworthy.
- rapidsai/cuml — GPU UMAP/t-SNE in the RAPIDS stack; use when you have CUDA and
  datasets large enough that CPU UMAP is the bottleneck.
- TorchDR/TorchDR — PyTorch reimplementation running the whole pipeline on GPU;
  use when you want UMAP-like output inside a Torch workflow.
- scikit-learn/scikit-learn — PCA and `TSNE` in one dependency; use when you want
  a linear baseline or want to avoid the Numba stack entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| arXiv paper | 2018-02 | McInnes & Healy publish the UMAP algorithm[^1]. |
| JOSS paper | 2018 | Software reference paper, `umap-learn` on PyPI[^6]. |
| 0.4 | 2020 | New neighbor search, plotting subpackage, sparse support. |
| 0.5 | 2021 | densMAP, Parametric UMAP, semi-supervised improvements[^4]. |
| Nature Primer | 2024 | Broad scientific introduction published in Nat Rev Methods Primers[^7]. |

## References

[^1]: McInnes, L., Healy, J., Melville, J. "UMAP: Uniform Manifold Approximation and Projection for Dimension Reduction." arXiv:1802.03426, 2018. https://arxiv.org/abs/1802.03426
[^2]: Chari, T., Pachter, L. "The Specious Art of Single-Cell Genomics." PLOS Computational Biology, 2023 — argues UMAP/t-SNE distort structure. https://doi.org/10.1371/journal.pcbi.1011288
[^3]: pynndescent — approximate nearest neighbor library used by UMAP. https://github.com/lmcinnes/pynndescent
[^4]: Narayan, A., Berger, B., Cho, H. "Assessing Single-Cell Transcriptomic Variability through Density-Preserving Data Visualization" (densMAP). Nature Biotechnology, 2021. https://doi.org/10.1038/s41587-020-00801-7
[^5]: UMAP docs, "Using UMAP for Clustering." https://umap-learn.readthedocs.io/en/latest/clustering.html
[^6]: McInnes, L., Healy, J., Saul, N., Grossberger, L. "UMAP." Journal of Open Source Software, 3(29):861, 2018. https://doi.org/10.21105/joss.00861
[^7]: Healy, J., McInnes, L. "Uniform manifold approximation and projection." Nature Reviews Methods Primers 4, 82 (2024). https://doi.org/10.1038/s43586-024-00363-x

## Tags

python, dimensionality-reduction, manifold-learning, visualization, machine-learning, embeddings, scikit-learn, numba, t-sne-alternative, single-cell, topological-data-analysis
