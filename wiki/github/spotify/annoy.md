# spotify/annoy

> A small, mmap-friendly approximate nearest neighbor library whose defining trick is treating the index as a static file that many processes can share.

[GitHub repo](https://github.com/spotify/annoy) ·
[License: Apache-2.0](https://github.com/spotify/annoy/blob/main/LICENSE)

## Overview

Annoy ("Approximate Nearest Neighbors Oh Yeah") is a C++ library with Python bindings for finding points in a vector space close to a query point. Erik Bernhardsson built it at Spotify during a 2013 Hack Week to serve music recommendations: after matrix factorization, every user and track becomes a vector in f-dimensional space, and Annoy answers "which vectors are nearest" over many millions of items[^1]. It has become one of the reference ANN libraries and a fixture of the `ann-benchmarks` suite[^2].

Its differentiator is not raw speed — several libraries are faster on pure query throughput — but its storage model. An Annoy index is a flat file that gets `mmap`ped into memory, so the OS page cache is shared across every process that opens the same file. You build the index once, ship the file to production, Hadoop jobs, or many worker processes, and each maps it in with near-zero load time and no per-process heap duplication[^1]. This makes Annoy well suited to read-heavy, memory-constrained, horizontally-scaled serving.

The tradeoff is baked into the design: the index is immutable. Once you call `build()`, you cannot add items, and there is no delete or update. Annoy is a batch-built, read-only structure — a snapshot, not a database. Development has slowed to maintenance; the codebase is mature and stable rather than actively evolving, with roughly 14k stars and infrequent commits as of 2025.

## Getting Started

```bash
pip install --user annoy      # Python bindings from PyPI
# C++: clone the repo and #include "annoylib.h" (header-only)
```

```python
from annoy import AnnoyIndex
import random

f = 40                              # vector dimensionality
t = AnnoyIndex(f, 'angular')        # angular | euclidean | manhattan | hamming | dot
for i in range(1000):
    t.add_item(i, [random.gauss(0, 1) for _ in range(f)])

t.build(10)                         # 10 trees; no more add_item after this
t.save('test.ann')

u = AnnoyIndex(f, 'angular')
u.load('test.ann')                  # mmap — effectively instant
print(u.get_nns_by_item(0, 1000))   # 1000 nearest neighbors of item 0
```

Item ids must be nonnegative integers; the index allocates memory for `max(id)+1` items because it assumes ids are dense `0..n-1`. Sparse or non-integer ids require your own external id map.

## Architecture / How It Works

Annoy builds a forest of random-projection trees[^3]. At each internal node it samples two points from the local subset and splits the space with the hyperplane equidistant between them; recursion continues until leaves hold a small number of points. Building `n_trees` independent trees this way produces a forest. A query descends all trees, collecting candidate leaves into a priority queue, then does exact distance re-ranking over the gathered candidates — approximate because it only inspects a bounded slice of the space.

Two knobs control the entire accuracy/cost curve:

- **`n_trees`** — set at build time. More trees means higher recall and a larger index file. It affects build time and disk/memory size, not query time (when `search_k` is held fixed).
- **`search_k`** — set per query. It caps how many nodes the search inspects; larger means slower but more accurate. If unset it defaults to `n * n_trees`. The two knobs are roughly independent, which is unusually clean for an ANN library and makes tuning tractable.

Supported metrics are angular (cosine), Euclidean, Manhattan, Hamming, and dot/inner-product. Angular distance is implemented as Euclidean distance of normalized vectors, `sqrt(2(1-cos(u,v)))`. Hamming packs vectors into 64-bit integers and uses popcount primitives with axis-aligned splits[^3]. Dot-product support maps inner-product space into a cosine-friendly space using the Bachrach et al. (Microsoft Research, 2014) transform, since inner product is not a true metric[^3].

The on-disk format is the whole point. Nodes are laid out contiguously so the file *is* the in-memory representation — no deserialization step. `load()` just `mmap`s it. With `prefault=True` the file is pulled into memory eagerly (`MAP_POPULATE`); with the default `prefault=False`, pages fault in lazily on demand, trading slower first queries for lower resident memory. `on_disk_build()` lets you build indexes too large for RAM by writing tree nodes to the file as they are produced.

## Production Notes

- **Immutability is the load-bearing constraint.** No add-after-build, no delete, no update. Any change to the corpus means a full rebuild and a file swap. Teams typically rebuild on a schedule and atomically replace the `.ann` file, reloading in serving processes. If you need incremental updates or deletes, Annoy is the wrong tool.
- **`max(id)+1` allocation is a footgun.** A single item with id 10,000,000 allocates space for ten million slots. Keep ids dense from zero and maintain your own mapping to real identifiers.
- **No bounds checking.** The README states plainly there is no bounds checking on values — malformed input can corrupt or crash rather than raise. Validate upstream.
- **Recall must be measured, not assumed.** With too few trees or too small a `search_k`, recall drops silently; there is no error, just worse neighbors. Validate against a brute-force ground truth on a sample before shipping, and re-check after any dimensionality or metric change.
- **Dimensionality.** Works best under ~100 dimensions but is reported to hold up surprisingly well up to ~1,000. Very high-dimensional data degrades as it does for all tree-based ANN methods.
- **`get_distance` semantics changed.** As of August 2016 it returns the actual distance, not the squared distance it returned previously — relevant when reading old code or comparing against archived values.
- **Not the throughput leader.** On `ann-benchmarks` Annoy is competitive but graph-based methods (HNSW) generally win on the recall-vs-QPS frontier. Choose Annoy for its operational model (shared mmap, tiny footprint, disk builds), not to top a benchmark.

## When to Use / When Not

**Use when:**
- Your corpus is static or rebuilt in batches, and read latency/footprint matter more than write flexibility.
- Many processes on a host serve the same index and you want them to share one memory-mapped copy.
- The index must be shipped as a portable file (to Hadoop, edge workers, containers) and loaded instantly.
- Memory is tight and the index may exceed RAM (`on_disk_build`).

**Avoid when:**
- You need to add, update, or delete vectors without a full rebuild.
- You need the highest recall-at-QPS — reach for an HNSW-based library.
- You need filtered search, metadata, persistence guarantees, or a query language — Annoy is a nearest-neighbor primitive, not a vector database.
- Your ids are sparse or non-integer and you cannot maintain an external map.

## Alternatives

- nmslib/hnswlib — HNSW graph index; higher recall-at-QPS and supports incremental inserts. Use when query quality/speed dominates and you can hold the graph in RAM.
- facebookresearch/faiss — broad ANN toolkit with GPU support, quantization, and billion-scale indexes. Use when you need scale, compression, or many index types in one library.
- google-research/google-research (ScaNN) — strong on the recall/speed frontier for large in-memory sets. Use when you want state-of-the-art CPU ANN and can adopt its build.
- spotify/voyager — Spotify's own newer HNSW-based library with Python/Java bindings, effectively the successor for use cases needing updates. Use when you want Spotify-lineage tooling but mutable indexes.
- qdrant/qdrant or milvus-io/milvus — full vector databases with filtering, persistence, and CRUD. Use when you need a service, not a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-04 | Built by Erik Bernhardsson during a Spotify Hack Week[^1]. |
| — | 2016-08 | `get_distance` changed from squared distance to actual distance. |
| — | 2016 | Dot-product metric added via Bachrach et al. transform[^3]. |
| — | ~2017–2019 | Hamming metric, on-disk build, multithreaded `build(n_jobs)`. |
| maintenance | 2020s | Stable, maintenance-mode; last push 2025-10[^4]. |

## References

[^1]: Annoy README — origin, Spotify usage, and mmap/shared-index design. https://github.com/spotify/annoy/blob/main/README.rst
[^2]: ann-benchmarks — comparative ANN benchmark suite including Annoy. https://github.com/erikbern/ann-benchmarks
[^3]: Annoy README, "How does it work" / "Tradeoffs" — random projection forest, metrics, Hamming and dot-product internals. https://github.com/spotify/annoy/blob/main/README.rst
[^4]: spotify/annoy repository metadata (stars, license, last push) via GitHub API, retrieved 2026-07. https://github.com/spotify/annoy

## Tags

c-plus-plus, python, approximate-nearest-neighbor, vector-search, mmap, locality-sensitive-hashing, embeddings, recommendations, machine-learning, read-only-index
