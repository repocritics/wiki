# facebookresearch/faiss

> A C++/Python library for similarity search and clustering of dense vectors — the ANN engine underneath much of the vector-search ecosystem.

[GitHub repo](https://github.com/facebookresearch/faiss) ·
[Official website](https://faiss.ai) ·
[License: MIT](https://github.com/facebookresearch/faiss/blob/main/LICENSE)

## Overview

Faiss (Facebook AI Similarity Search) is a library for nearest-neighbor search over large collections of dense vectors, developed primarily at Meta's Fundamental AI Research group and open-sourced in February 2017[^1]. It answers one narrow question extremely well: given a query vector, find the vectors in a stored set that are closest under L2 distance or inner product (and, on normalized vectors, cosine). It is written in C++ with complete Python/numpy wrappers generated via SWIG, and ships GPU implementations for the most useful algorithms[^2].

The defining characteristic of Faiss is that it is a *library of index structures*, not a database. Instead of one search algorithm it offers a menu of index types that trade off against each other along several axes: search latency, recall, memory per vector, training time, and add time[^3]. Exact flat search, inverted-file partitioning (IVF), product quantization (PQ) and its additive-quantization relatives, and graph indexes (HNSW, NSG) can be composed. Choosing and tuning the right combination for a dataset is the real work of using Faiss, and the library deliberately pushes that decision onto the caller.

That design is also its central tension. Faiss gives you exact control over the memory/recall/latency frontier and scales to billions of vectors on a single machine, but it provides none of the surrounding machinery a production system usually wants: persistence beyond flat-file dumps, metadata filtering, concurrent writers, replication, or CRUD durability. Those are left to the layer above — which is exactly why several vector databases (Milvus, among others) embed Faiss rather than replace it.

## Getting Started

```bash
# CPU build via conda (the primary distribution channel)
conda install -c pytorch faiss-cpu
# GPU build (CUDA)
conda install -c pytorch faiss-gpu
# or community-maintained pip wheels (CPU)
pip install faiss-cpu
```

```python
import numpy as np
import faiss

d = 128                          # vector dimension
nb, nq = 100_000, 10             # database and query counts
rng = np.random.default_rng(0)
xb = rng.random((nb, d), dtype=np.float32)   # MUST be float32, C-contiguous
xq = rng.random((nq, d), dtype=np.float32)

index = faiss.IndexFlatL2(d)     # exact brute-force baseline
index.add(xb)                    # no training needed for a flat index
D, I = index.search(xq, k=5)     # D=distances, I=neighbor ids, shape (nq, 5)

# Compressed, partitioned index for scale:
quantizer = faiss.IndexFlatL2(d)
ivf = faiss.IndexIVFPQ(quantizer, d, 1024, 16, 8)  # 1024 cells, 16 subquantizers, 8 bits
ivf.train(xb)                    # IVF/PQ REQUIRE training on representative data
ivf.add(xb)
ivf.nprobe = 16                  # cells probed per query — the recall/latency knob
D, I = ivf.search(xq, k=5)
```

The `index_factory` shorthand builds the same structures from a string, e.g. `faiss.index_factory(d, "IVF1024,PQ16")` or `"OPQ16,IVF4096,PQ16"`.

## Architecture / How It Works

Everything centers on the `Index` base class, which exposes `train`, `add`, and `search`. Concrete index types implement that interface with different internal representations:

- **Flat** (`IndexFlatL2`, `IndexFlatIP`) — stores raw float32 vectors and scans all of them. Exact, no training, but O(n·d) per query and 4·d bytes per vector in RAM.
- **IVF** — a coarse quantizer (usually k-means, i.e. an `IndexFlat` of centroids) partitions the space into Voronoi cells; at query time only `nprobe` nearby cells are scanned. This is the primary speed lever and the primary recall risk: too few probes miss true neighbors near cell boundaries.
- **PQ / additive quantization** — compress each vector into a short code (product quantization splits the vector into sub-vectors, each replaced by a codebook index). Distances are computed in the compressed domain via lookup tables, so PQ indexes hold billions of vectors in memory at the cost of approximate distances. Residual and local-search quantizers refine this.
- **Graph indexes** (`IndexHNSW`, `IndexNSG`) — build a navigable-small-world or NSG graph over the raw vectors. High recall and fast search, but memory-heavy (they keep the full vectors plus graph links) and slow to build.

Indexes compose: `IVFPQ` puts PQ codes inside IVF cells; `OPQ` is a learned rotation applied before PQ; `IndexIDMap` wraps any index to attach arbitrary 64-bit ids; `IndexPreTransform` chains preprocessing. Serialization is `write_index` / `read_index` to a single flat file.

The Python layer is a thin SWIG wrapper over the C++ objects — arrays cross the boundary as raw pointers, which is why inputs must be `float32` and C-contiguous or results are silently wrong. The GPU path (`GpuIndexFlatL2`, `index_cpu_to_gpu`) mirrors the CPU indexes and can be a drop-in replacement; recent versions can also delegate to NVIDIA's cuVS kernels[^2].

## Production Notes

**Faiss is not a database — this is the most common misunderstanding.** No metadata filtering, no transactional persistence, no replication, no built-in sharding. Deletions are limited: `remove_ids` works on some index types (IDMap, IVF) and is unsupported or expensive on others; many compressed indexes are effectively append-only. If you need filtered search or durable CRUD, put Faiss behind your own service layer or use a vector database that wraps it.

**Training is mandatory and data-sensitive for IVF/PQ.** Flat indexes need no training, but IVF and PQ must be `train`ed on a representative sample before `add`. Rules of thumb from the docs: enough training points to populate centroids (tens of points per centroid at minimum), drawn from the same distribution as the data. Training on unrepresentative or too-small data quietly degrades recall.

**Recall/latency tuning lives in a few parameters.** `nprobe` (IVF cells probed), `efSearch` (HNSW), and the choice of code size dominate. There is no single correct setting — you sweep them against a labeled recall benchmark for your data. Faiss ships tooling for this (`benchs/`, `faiss.contrib`) but does not do it for you.

**Thread safety is read-mostly.** Concurrent `search` calls on an unchanged index are safe; concurrent `add`/`train` with `search` is not. Faiss uses OpenMP internally and links a BLAS implementation, so it competes with your own thread pool — oversubscription (OpenMP threads × BLAS threads × your workers) is a classic latency footgun. Control it with `omp_set_num_threads` and BLAS env vars.

**Memory is the real budget.** A flat index is 4·d bytes/vector; PQ can cut that by 10–100× at a recall cost. Estimate before you build — an IVF index also stores coarse centroids and inverted lists. On GPU, index size is bounded by device memory, and float16 storage is often needed to fit.

**Serialization compatibility.** Index files are versioned and generally forward-compatible across releases, but a file written by a newer Faiss may not load on an older one. Pin the version that reads your persisted indexes rather than assuming cross-version portability for exotic index types.

## When to Use / When Not

**Use when:**
- You need approximate nearest-neighbor search over millions-to-billions of vectors and want explicit control of the memory/recall/latency tradeoff.
- You are building the retrieval core of a RAG, dedup, recommendation, or clustering pipeline and will own the surrounding service layer yourself.
- You do offline or batch index builds and value raw performance and a mature quantization toolkit over operational convenience.
- You need GPU-accelerated k-means or k-selection, not just search.

**Avoid when:**
- You want a turnkey vector database with filtering, metadata, persistence, and horizontal scaling — use a real vector DB instead of reimplementing one around Faiss.
- Your dataset is small enough (tens of thousands) that exact flat search is instant and index tuning buys nothing.
- You need frequent per-record deletes/updates with durability guarantees.
- Your team lacks the appetite to benchmark and tune index parameters against recall targets.

## Alternatives

- spotify/annoy — simpler tree-based ANN with mmap-friendly static indexes; use when you want a read-only index and minimal tuning over Faiss's flexibility.
- nmslib/hnswlib — lightweight header-only HNSW; use when you only need one graph index and want a smaller dependency than Faiss.
- google/scann — Google's ANN library with strong anisotropic quantization; use when quantization recall at scale is the priority and a narrower index menu is acceptable.
- milvus-io/milvus — full vector database that embeds Faiss-class engines; use when you need filtering, persistence, and distribution rather than a raw library.
- qdrant/qdrant — Rust vector database with payload filtering; use when you want an operable service out of the box instead of building one.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-02 | Open-sourced by FAIR alongside the billion-scale GPU search paper[^1][^4]. |
| GPU + binary | 2017–2018 | GPU indexes, binary (Hamming) indexes, and additive quantization added over time. |
| 1.7.x | 2021–2023 | Long-lived line; HNSW/NSG maturation and broad platform wheels. |
| 1.8.0 | 2024 | Released around the "Faiss library" paper describing the design[^3]. |
| 1.9.x–1.11.x | 2024–2025 | NVIDIA cuVS GPU backend integration and ongoing quantization/SIMD work[^2]. |

## References

[^1]: Johnson, Douze, Jégou, "Billion-scale similarity search with GPUs" — arXiv:1702.08734, Feb 2017. https://arxiv.org/abs/1702.08734
[^2]: Faiss README and INSTALL — GPU support via CUDA/ROCm and optional NVIDIA cuVS backend. https://github.com/facebookresearch/faiss/blob/main/README.md
[^3]: Douze et al., "The Faiss library" — arXiv:2401.08281, Jan 2024. https://arxiv.org/abs/2401.08281
[^4]: Faiss wiki — indexing structures, factory strings, and tuning guides. https://github.com/facebookresearch/faiss/wiki

## Tags

c-plus-plus, python, vector-search, nearest-neighbor, similarity-search, embeddings, ann, quantization, gpu, clustering, information-retrieval, machine-learning
