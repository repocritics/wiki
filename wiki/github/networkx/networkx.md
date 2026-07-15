# networkx/networkx

> Pure-Python library for building, analyzing, and drawing graphs — the default graph toolkit in scientific Python, chosen for reach over raw speed.

[GitHub repo](https://github.com/networkx/networkx) ·
[Official website](https://networkx.org) ·
[License: BSD-3-Clause](https://github.com/networkx/networkx/blob/main/LICENSE.txt)

## Overview

NetworkX is a Python package for creating, manipulating, and studying the structure and dynamics of complex networks. It originated in 2002–2004 at Los Alamos National Laboratory (Aric Hagberg, Dan Schult, Pieter Swart) and has become the de facto graph library of the scientific-Python stack, sitting alongside NumPy, SciPy, pandas, and Matplotlib[^1]. It ships four core graph types — `Graph`, `DiGraph`, `MultiGraph`, `MultiDiGraph` — plus a very wide catalog of algorithms: shortest paths, centrality, connectivity, flow, matching, community detection, clustering, and dozens of graph generators.

Its defining design choice is that it is pure Python with no required dependencies. Graphs are stored as a dict-of-dicts-of-dicts adjacency structure, and nodes can be any hashable Python object — a string, a tuple, a custom class. This makes NetworkX exceptionally easy to learn, script against, and integrate, and makes arbitrary per-node/per-edge attributes trivial.

The same choice is its central tradeoff: the pure-Python representation is memory-heavy and slow on large graphs. NetworkX is excellent for graphs up to the low millions of edges, for teaching, prototyping, and analysis where developer time dominates. It is the wrong tool when you need to run heavy algorithms on tens of millions of edges or more — for that, people reach for compiled libraries or, increasingly, NetworkX's own pluggable backends (below).

## Getting Started

```shell
pip install networkx            # pure Python, no hard deps
pip install networkx[default]   # + numpy, scipy, matplotlib, pandas
```

```python
import networkx as nx

G = nx.Graph()
G.add_edge("A", "B", weight=4)
G.add_edge("B", "D", weight=2)
G.add_edge("A", "C", weight=3)
G.add_edge("C", "D", weight=4)

print(nx.shortest_path(G, "A", "D", weight="weight"))   # ['A', 'B', 'D']
print(nx.betweenness_centrality(G))                      # {'A': ..., 'B': ...}
```

Nodes are any hashable object and attributes are free-form dicts, so `G.nodes[n]` and `G.edges[u, v]` behave like ordinary Python dictionaries.

## Architecture / How It Works

The data model is a nested-dict adjacency map. For a `Graph`, `G._adj[u][v]` is the attribute dict of edge `(u, v)`, mirrored so `G._adj[v][u]` is the same dict; `G._node[u]` holds node attributes. This gives O(1) node/edge lookup and insertion at the cost of Python object overhead per node and per edge. Views (`G.nodes`, `G.edges`, `G.adj`, `G.degree`) are lazy, read-only wrappers over these dicts, added in the 2.0 rewrite to replace the mutable methods of the 1.x era[^2].

Algorithms are plain Python functions in `networkx.algorithms`, operating on the graph interface rather than on the internal dicts, which is why custom graph subclasses interoperate cleanly. Numeric-heavy routines (spectral methods, some centralities, linear-algebra conversions) optionally use NumPy/SciPy when installed, via `to_numpy_array` / `to_scipy_sparse_array`.

The most consequential recent architectural change is the **backend dispatch system** introduced across the 3.x line[^3]. Public functions are decorated so a call like `nx.betweenness_centrality(G)` can be routed to an alternative implementation registered through a Python entry point — for example `nx-cugraph` (NVIDIA GPU), `graphblas-algorithms`, or `nx-parallel`. Backend selection is controlled by `nx.config.backend_priority`, the `backend=` keyword, or the `NETWORKX_BACKEND_PRIORITY` environment variable. When a backend does not implement a function, NetworkX transparently falls back to its own pure-Python version. This lets code stay written against the NetworkX API while swapping the compute engine underneath — the project's answer to its performance ceiling without forcing a rewrite.

## Production Notes

- **Performance is the recurring footgun.** Pure-Python iteration means algorithms with poor asymptotic complexity (all-pairs shortest paths, exact betweenness) become impractical well before you exhaust memory. Profile before assuming NetworkX can carry a workload; a graph that fits in RAM is not the same as one you can run global algorithms on.
- **Memory overhead is large.** Every node and edge is a Python object plus dict entries. A rough field heuristic is on the order of hundreds of bytes to ~1 KB per edge depending on attributes — orders of magnitude more than a compact CSR array. Convert to a SciPy sparse matrix for large linear-algebra work.
- **Backends are not a drop-in for everything.** `nx-cugraph` and friends implement a subset of functions; unsupported calls silently fall back to slow Python unless you set backend priority to raise. Verify which functions your pipeline actually uses are accelerated before promising a speedup.
- **Reproducibility.** Many generators and approximation algorithms take a `seed` argument; omit it and results vary run to run. Pin `seed` for anything tested or published.
- **Visualization is minimal.** `nx.draw` is a thin Matplotlib helper meant for small graphs and quick looks, not production rendering. For anything sizable, export (GraphML, GEXF, edge lists) to Gephi, Graphviz, or a JS library.
- **Upgrade friction.** The 1.x → 2.0 jump (2017) was a broad, breaking API rewrite — mutable `G.node`/`G.edge` became read-only views `G.nodes`/`G.edges`, and many function signatures changed[^2]. The 3.0 release (2023) removed long-deprecated 2.x APIs. Recent releases move the minimum Python version forward fairly aggressively, so pin `networkx` and your Python together in CI.

## When to Use / When Not

**Use when:**
- You are analyzing graphs from thousands up to a few million edges and value flexibility over throughput.
- You need rich, arbitrary node/edge attributes and hashable-object nodes.
- You are teaching, prototyping, or doing exploratory network science.
- You want a wide algorithm catalog with a stable, well-documented API.

**Avoid when:**
- Your graph is tens of millions of edges or larger and you need global algorithms — reach for a compiled or GPU library (and consider a NetworkX backend to keep the API).
- You are latency- or memory-constrained in production serving.
- You mainly need graph storage/traversal with persistence and queries — that is a graph database, not an in-memory analysis library.

## Alternatives

- igraph/python-igraph — C core with Python bindings; much faster and lighter on large graphs, at the cost of a less Pythonic API. Use when NetworkX hits its speed/memory ceiling.
- Qiskit/rustworkx — Rust-backed graph library with a NetworkX-like surface; use when you want compiled speed and a familiar API in one package.
- rapidsai/cugraph — GPU-accelerated graph analytics; also usable as the `nx-cugraph` NetworkX backend. Use for very large graphs on NVIDIA hardware.
- graph-tool (Tiago Peixoto) — C++/Boost core, strong statistical-inference and drawing features; use for heavy analysis and publication-grade visuals, though it is harder to install.
- snap-stanford/snap — C++ library for very large network analysis; use for scale when a lower-level API is acceptable.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010-01 | First 1.x series release; established the core API. |
| 2.0 | 2017-09 | Major breaking rewrite — read-only views, mutable-method removal[^2]. |
| 2.5 | 2020-08 | Continued 2.x line; broad algorithm additions. |
| 3.0 | 2023-01 | Removed deprecated 2.x APIs; Python 3-only cleanup[^4]. |
| 3.2 | 2023-10 | Backend dispatch / entry-point plugin system matured[^3]. |
| 3.x | 2024–2026 | Ongoing releases; expanding backend support (cugraph, parallel), moving minimum Python forward. |

## References

[^1]: Hagberg, Schult, Swart, "Exploring Network Structure, Dynamics, and Function using NetworkX," Proceedings of the 7th Python in Science Conference (SciPy 2008). https://networkx.org/documentation/stable/citing.html
[^2]: NetworkX 2.0 migration guide. https://networkx.org/documentation/stable/release/migration_guide_from_1.x_to_2.0.html
[^3]: NetworkX backends and configs reference. https://networkx.org/documentation/stable/reference/backends.html
[^4]: NetworkX release notes. https://networkx.org/documentation/stable/release/index.html

## Tags

python, graph-theory, network-analysis, graph-algorithms, centrality, scientific-python, pure-python, data-science, complex-networks, visualization
