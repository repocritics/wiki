# nmslib/hnswlib

> Header-only C++/Python implementation of the HNSW graph for in-memory approximate nearest-neighbor search.

[GitHub repo](https://github.com/nmslib/hnswlib) ·
[Algorithm parameters](https://github.com/nmslib/hnswlib/blob/master/ALGO_PARAMS.md) ·
[License: Apache-2.0](https://github.com/nmslib/hnswlib/blob/master/LICENSE)

## Overview

hnswlib is a single-purpose library: it builds and queries a Hierarchical
Navigable Small World (HNSW) graph for approximate k-nearest-neighbor search over
dense float32 vectors. It is the reference implementation from one of the HNSW
paper's authors (Yury Malkov)[^1], spun out of the larger nmslib project as a
lean, dependency-free alternative focused on this one algorithm. The C++ core is
header-only (no build step beyond including it), and the Python bindings are a
thin pybind11 wrapper.

Its defining characteristic is deliberate minimalism. There is no quantization,
no compression, no GPU path, no disk-backed index, and no distributed layer —
the entire graph and all raw vectors live in RAM. That constraint is also its
appeal: the code is small enough to read in an afternoon, easy to vendor, and
fast enough that it became the embedded ANN engine inside a generation of vector
databases and RAG tooling (Chroma bundles it; early Qdrant, Weaviate, and Milvus
builds used it before writing their own). The tension is that this simplicity
stops scaling exactly where memory does: a billion 768-dim vectors is terabytes
of float32, which is where Faiss's compressed indexes take over.

The repository is mature and low-velocity rather than abandoned — the last
release cadence is roughly annual and the API has been stable for years. Much of
the ecosystem now consumes HNSW through downstream reimplementations, so hnswlib
functions as much as a canonical spec as an actively-featured library.

## Getting Started

```bash
pip install hnswlib          # Python bindings (pulls a C++ build toolchain)
# C++: no install — copy hnswlib/ headers and #include "hnswlib/hnswlib.h"
```

```python
import hnswlib
import numpy as np

dim, n = 128, 10_000
data = np.float32(np.random.random((n, dim)))
ids = np.arange(n)

p = hnswlib.Index(space="l2", dim=dim)      # space: l2 | ip | cosine
p.init_index(max_elements=n, ef_construction=200, M=16)
p.add_items(data, ids)                       # incremental; can be called repeatedly

p.set_ef(50)                                 # query-time accuracy knob; must be > k
labels, distances = p.knn_query(data, k=5)

p.save_index("index.bin")                    # NOTE: ef is not persisted — re-set after load
```

## Architecture / How It Works

HNSW is a multi-layer proximity graph. The bottom layer contains every element;
each higher layer is an exponentially sparser subset holding long-range links.
A search enters at the single top-layer entry point and greedily walks toward the
query vector, descending a layer each time it reaches a local minimum, until it
does a final beam search on the dense bottom layer. This gives roughly
logarithmic search complexity while keeping high recall.

Three parameters govern the tradeoff space[^2]:

- **`M`** — max outgoing connections per node per layer (the bottom layer allows
  `2*M`). Higher `M` improves recall on high-dimensional data but raises memory
  linearly and slows construction.
- **`ef_construction`** — size of the candidate list while inserting. Higher
  means a better-connected graph at higher build cost.
- **`ef`** — size of the candidate list at query time. The primary recall/latency
  dial, tunable per query without rebuilding.

Because it is header-only, the graph, distance functions (SIMD-accelerated L2 /
inner-product kernels for SSE/AVX), and the visited-list pool all compile
directly into the host binary. Vectors are stored inline in a flat contiguous
block alongside their per-node link lists, which is why the index is a single
`mmap`-friendly binary blob but also why memory is dominated by raw float32
storage plus `~M * sizeof(int)` of link overhead per element. Custom distance
functions are a C++-only feature; the Python side is fixed to `l2`, `ip`, and
`cosine` (where `ip` is not a true metric, and `cosine` stores vectors
pre-normalized).

## Production Notes

**Capacity is preallocated.** `init_index(max_elements=...)` fixes the index
size; exceeding it throws. `resize_index()` exists but reallocates the whole
block and is not thread-safe with reads or writes — treat capacity as a
provisioning decision, not something to grow per-insert.

**Deletions are soft.** `mark_deleted()` only hides an element from results; its
memory is not reclaimed. To bound index growth you must build with
`allow_replace_deleted=True` and insert with `replace_deleted=True` so new
elements reuse deleted slots[^3]. Updating an existing label is supported but
slower than a fresh insert.

**The threading model is specific and easy to get wrong.** Concurrent
`add_items` calls are safe with each other; concurrent `knn_query` calls are safe
with each other; but mixing inserts and queries concurrently is not safe, and
neither is safe against `resize_index`. In Python, filtered search is
bottlenecked by the GIL — the docs recommend `num_threads=1` when passing a
`filter`, which erases the parallelism you might expect.

**`ef` is not serialized.** After `load_index`, the query-time `ef` resets and
must be set again manually — a silent recall regression if forgotten. Pickling
the Python index is supported but is not thread-safe against concurrent
`add_items`.

**It is RAM-only and single-machine.** There is no sharding, replication, or
on-disk index. Sizing is essentially `n * dim * 4 bytes` plus link overhead; plan
host memory accordingly and offload to Faiss (product quantization) or a
disk-based index (DiskANN) once vectors no longer fit. There is also no built-in
incremental persistence — you re-`save_index` the whole file.

**Provenance caveat.** The dynamic-update algorithm in this repo is covered by a
US patent held by the contributors (Sharma, Tayal, Malkov)[^4]; the code is
Apache-2.0, but organizations with strict IP-review processes occasionally flag
this.

## When to Use / When Not

**Use when:**
- Your vector set fits in RAM and you want maximum recall-per-latency from a
  small, auditable, header-only dependency you can vendor.
- You need incremental inserts, updates, and deletions on a live index rather
  than a batch-built static one.
- You're embedding ANN inside another C++/Python system and don't want to pull in
  a heavy framework.

**Avoid when:**
- You're at billion-scale or memory-bound — you need quantization/compression
  (Faiss) or a disk-based graph (DiskANN).
- You need GPU acceleration, a distributed/sharded cluster, or built-in
  persistence and replication — use a full vector database.
- You require exotic distance metrics from Python, or heavy concurrent
  read-write traffic against a single index.

## Alternatives

- facebookresearch/faiss — use instead when you need vector compression (PQ/OPQ),
  GPU search, or billion-scale indexes; Faiss also has its own HNSW.
- spotify/annoy — use instead when you want a static, `mmap`-able tree index with
  a tiny memory footprint and don't need incremental updates.
- google-research/google-research (ScaNN) — use instead when you want top-tier
  recall/latency via anisotropic quantization on large static datasets.
- unum-cloud/usearch — use instead when you want a modern HNSW with quantization,
  more language bindings, and richer SIMD than hnswlib exposes.
- nmslib/nmslib — use instead when you need non-metric or exotic distance spaces
  beyond L2/IP/cosine.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-07 | Split out of nmslib as a standalone header-only HNSW[^1]. |
| 0.7.0 | ~2023 | Label filtering (C++ and Python), delete-slot replacement, thread-safety fixes for updates/insertion. |
| 0.8.0 | ~2024 | Multi-vector document search and epsilon search (C++ only); stat aggregation off by default for faster multi-threaded search. |
| 0.9.0 | ~2025 | Brute-force filtered-search correctness fixes; throw when fewer than k elements exist. |

## References

[^1]: Malkov & Yashunin, "Efficient and robust approximate nearest neighbor
search using Hierarchical Navigable Small World graphs," IEEE TPAMI vol. 42
(2018). https://arxiv.org/abs/1603.09320
[^2]: hnswlib parameter guide (ALGO_PARAMS.md). https://github.com/nmslib/hnswlib/blob/master/ALGO_PARAMS.md
[^3]: hnswlib README — deletion and element replacement semantics. https://github.com/nmslib/hnswlib#readme
[^4]: "Dynamic Updates For HNSW, Hierarchical Navigable Small World Graphs," US Patent application 15/929,802 (Sharma, Tayal, Malkov), noted in the project README.

## Tags

cpp, python, approximate-nearest-neighbors, hnsw, vector-search, similarity-search, embeddings, header-only, ann, machine-learning
