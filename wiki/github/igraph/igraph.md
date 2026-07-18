# igraph/igraph

> The C core of the igraph network-analysis stack — the engine behind R's `igraph` package and `python-igraph`, built for graphs with millions of edges on a single machine.

[GitHub repo](https://github.com/igraph/igraph) ·
[Official website](https://igraph.org) ·
[License: GPL-2.0](https://github.com/igraph/igraph/blob/main/COPYING)

## Overview

igraph is a C library for graph theory and complex network analysis, started by Gábor Csárdi and Tamás Nepusz and announced in a 2006 paper that has since become one of the most-cited references in network science[^1]. It bundles a large algorithm catalog — centrality measures, community detection (Leiden, Louvain, Infomap, spinglass), shortest paths, graph isomorphism, random graph models, motif counting, and force-directed layouts — behind a single C API.

The repository's ~2.0k stars understate its reach: most users never touch the C API. They consume igraph through the R interface (`igraph/rigraph`), the Python interface (`igraph/python-igraph`), or Mathematica (IGraph/M)[^2], and those interfaces are what made igraph a default tool in computational social science, bioinformatics, and ecology. The C core exists so that all three frontends share one tested, fast implementation.

The defining tension: igraph is a hand-written C library in a domain where competitors use C++ templates, OpenMP, or GPUs. C buys portability and easy binding to any host language, but it means manual memory management, its own container types (`igraph_vector_t`, `igraph_matrix_t`), and largely single-threaded execution. After a slow period in the mid-2010s, development was revitalized around 2020 with dedicated grant funding, and the project has been actively modernizing the API since (277 open issues, last push 2026-07)[^3].

## Getting Started

```bash
# macOS
brew install igraph
# or build from source (CMake)
git clone https://github.com/igraph/igraph.git
cd igraph && mkdir build && cd build
cmake .. && cmake --build . && sudo cmake --install .
```

```c
/* example.c — compile: cc example.c $(pkg-config --cflags --libs igraph) */
#include <igraph.h>
#include <stdio.h>

int main(void) {
    igraph_t graph;
    igraph_vector_int_t degrees;

    igraph_rng_seed(igraph_rng_default(), 42);
    igraph_erdos_renyi_game_gnp(&graph, 1000, 5.0 / 1000,
                                IGRAPH_UNDIRECTED, IGRAPH_NO_LOOPS);

    igraph_vector_int_init(&degrees, 0);
    igraph_degree(&graph, &degrees, igraph_vss_all(),
                  IGRAPH_ALL, IGRAPH_NO_LOOPS);
    printf("max degree: %" IGRAPH_PRId "\n",
           igraph_vector_int_max(&degrees));

    igraph_vector_int_destroy(&degrees);
    igraph_destroy(&graph);
    return 0;
}
```

## Architecture / How It Works

The core data structure is an **indexed edge list**, not adjacency lists: vertices are dense integers `0..n-1`, edges live in flat vectors with sorted index structures for fast incidence queries. This is cache-friendly and memory-lean, but it has a consequence users hit constantly: **deleting vertices renumbers the remaining vertex IDs** to keep them contiguous, so any IDs you held before the deletion are invalid after it[^4]. Stable identity must be carried in attributes, not IDs.

Other structural choices:

- **Own container types.** `igraph_vector_t`, `igraph_vector_int_t`, `igraph_matrix_t` etc. replace raw C arrays throughout the API. Every `_init` must be paired with `_destroy`; valgrind is your friend.
- **Error handling** uses return codes plus an `IGRAPH_FINALLY` cleanup stack. The default error handler prints and aborts the process — any program embedding igraph as a library must install its own handler (`igraph_set_error_handler`) to get recoverable errors[^5].
- **Attributes are opt-in at the C level.** The C core only ships a plain attribute table (`igraph_cattribute_table`) that you must register explicitly; the rich attribute semantics people know from R/Python are implemented in those interfaces, not in the core.
- **Vendored dependencies.** Isomorphism (BLISS), power-law fitting (plfit), PageRank (PRPACK), and ARPACK-derived eigensolver code are bundled; external BLAS/LAPACK/GLPK can be linked in via CMake options.
- **0.10 API overhaul** (2022): `igraph_integer_t` became 64-bit by default, lifting the ~2 billion edge ceiling, with wide-ranging signature breaks as the project cleans the API toward a 1.0[^6].

## Production Notes

- **Batch your mutations.** Adding edges one at a time is quadratic in practice; build an edge vector and add it in one `igraph_add_edges` call. The library is optimized for analyze-mostly workloads, not high-churn dynamic graphs.
- **Mostly single-threaded.** Most algorithms use one core. igraph can be built thread-safe so that different threads work on different graphs, but there is no internal parallelism to speak of; for multi-core or billion-edge work, look at NetworKit, graph-tool, or cuGraph.
- **The vertex-ID renumbering footgun** (see above) is the most common source of silent logic bugs in downstream code, including via the R and Python wrappers.
- **API stability is not guaranteed pre-1.0.** 0.9 and especially 0.10 broke large numbers of function signatures. If you bind the C API directly, pin a minor version and read the CHANGELOG before upgrading[^3].
- **GPL-2.0 is a real constraint.** Linking igraph into a proprietary, distributed product effectively requires releasing that product under GPL terms. The permissively-licensed alternatives below matter for commercial embedding.
- **Long-running calls are interruptible** via a user-installed interruption handler — wire this up in interactive hosts, or a misjudged exact-isomorphism or community-detection call will pin a core indefinitely.

## When to Use / When Not

**Use when:**
- You work in R or Python and need fast classical network analysis — the interfaces are the intended entry point.
- You are embedding graph algorithms in a C/C++ application and want one portable, dependency-light library with a broad algorithm catalog.
- Your graphs fit in one machine's RAM (up to hundreds of millions of edges with the 64-bit build).

**Avoid when:**
- You need multi-core or GPU throughput on huge graphs — NetworKit, graph-tool, or cuGraph parallelize; igraph mostly does not.
- Your workload is a high-churn dynamic graph or an OLTP-style graph store — use a graph database, not an analysis library.
- You are shipping proprietary software and cannot accept GPL-2.0.
- You want maximum Python ergonomics and your graphs are small — NetworkX is slower but far more comfortable to hack on.

## Alternatives

- networkx/networkx — pure-Python; use instead when developer ergonomics and ecosystem beat raw speed on small-to-medium graphs.
- count0/graph-tool — Python over C++/Boost with OpenMP; use when you need parallel algorithms and statistical inference (SBMs) and can stomach the heavy build.
- networkit/networkit — use for parallel analysis of very large graphs from Python with a friendlier license (MIT).
- snap-stanford/snap — C++ library from Stanford; use for research workloads already standardized on SNAP datasets and tooling.
- rapidsai/cugraph — use when graphs and budget justify NVIDIA GPUs and you live in the RAPIDS ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2006-01 | First public release, alongside the Csárdi & Nepusz paper[^1]. |
| 0.6 | 2012 | Major release; R interface switched to 1-based vertex indexing. |
| 0.8 | 2020 | Development revival after a slow maintenance period. |
| 0.9 | 2021-02 | CMake build system; large deprecation sweep[^3]. |
| 0.10 | 2022-09 | 64-bit `igraph_integer_t` by default; broad breaking API cleanup toward 1.0[^6]. |

## References

[^1]: Csárdi, G., & Nepusz, T. (2006). "The igraph software package for complex network research." InterJournal, Complex Systems, 1695. https://igraph.org
[^2]: igraph README — R, Python, and Mathematica interfaces. https://github.com/igraph/igraph#readme
[^3]: igraph CHANGELOG. https://github.com/igraph/igraph/blob/main/CHANGELOG.md
[^4]: igraph C reference manual, "Adding and deleting vertices and edges". https://igraph.org/c/doc/
[^5]: igraph C reference manual, "Error handling". https://igraph.org/c/doc/igraph-Error.html
[^6]: igraph 0.10 release notes (CHANGELOG entry, 2022-09). https://github.com/igraph/igraph/blob/main/CHANGELOG.md

## Tags

c, graph-theory, network-analysis, graph-algorithms, complex-networks, community-detection, scientific-computing, library, r-ecosystem, python-bindings
