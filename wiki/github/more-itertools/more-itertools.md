# more-itertools/more-itertools

> The unofficial standard library for Python iterables — 130+ composable routines that pick up where `itertools` stops.

[GitHub repo](https://github.com/more-itertools/more-itertools) ·
[Official docs](https://more-itertools.rtfd.io) ·
[License: MIT](https://github.com/more-itertools/more-itertools/blob/master/LICENSE)

## Overview

more-itertools is a pure-Python, zero-dependency collection of iterator utilities, started by Erik Rose in 2012 and maintained in recent years primarily by Bo Bayles[^1]. Where the standard library's `itertools` ships a deliberately minimal core plus a "recipes" section of copy-paste snippets in its docs[^2], more-itertools packages those recipes as importable functions and adds well over a hundred of its own: `chunked`, `windowed`, `peekable`, `one`, `bucket`, `split_when`, `map_reduce`, `distribute`, and so on. Its 4k stars understate its reach — it sits in the transitive dependency tree of a large fraction of the Python ecosystem and is among the most-downloaded packages on PyPI.

The project's defining dynamic is its role as an **incubator for the standard library**. `itertools.pairwise` (Python 3.10) and `itertools.batched` (Python 3.12) both existed in more-itertools first[^3]. The tradeoff cuts both ways: you get tomorrow's stdlib today, but as functions graduate, you accumulate code that has a faster C-implemented stdlib equivalent on newer interpreters, and naming/signature drift between the two namespaces becomes something to track.

The library is actively maintained (last push 2026-07-17), with new function categories still landing — the README now lists a concurrency group (`concurrent_tee`, `serialize`, `synchronized`) alongside the classic grouping/windowing/selecting families.

## Getting Started

```bash
pip install more-itertools
```

```python
from more_itertools import chunked, windowed, one, peekable

list(chunked([1, 2, 3, 4, 5], 2))   # [[1, 2], [3, 4], [5]]
list(windowed([1, 2, 3, 4], 2))     # [(1, 2), (2, 3), (3, 4)]

one([42])        # 42 — raises ValueError if the iterable has 0 or 2+ items

p = peekable(iter("abc"))
p.peek()         # 'a' — look ahead without consuming
next(p)          # 'a'
```

Everything is importable flat from `more_itertools`; there is no configuration and no runtime dependency.

## Architecture / How It Works

The package is essentially two modules re-exported through one namespace[^1]:

- **`more_itertools.more`** — the original functions. Most are thin, lazy compositions over `itertools` primitives (`tee`, `islice`, `chain`, `groupby`); stateful behaviors are classes (`peekable`, `seekable`, `bucket`, `numeric_range`) that wrap an iterator and add caching or indexing.
- **`more_itertools.recipes`** — importable implementations of the recipes published in the CPython `itertools` documentation[^2], kept roughly in sync with upstream.

Design idioms worth knowing:

- **Lazy by default, but not uniformly.** Most functions yield incrementally, but several buffer or materialize: `seekable` caches every element it has seen (unbounded memory unless you pass `maxlen`), `bucket` buffers items for keys you haven't consumed yet, `distribute` uses `tee` (buffers up to the widest consumption gap between children), and `divide` consumes the entire input up front. The docstrings state this per function; the function name does not.
- **Exceptions as API.** `one`, `only`, `strictly_n`, and `exactly_n` exist to turn "I expected exactly N items" into a raised error instead of a silent truncation — useful as lightweight validation at data boundaries.
- **Typing via stub files.** Annotations ship as `.pyi` stubs rather than inline hints, keeping the runtime source compact while remaining fully typed for checkers.

There is no C extension. Every function pays Python-level per-item overhead, which is the main thing separating it from the stdlib's C-implemented `itertools`.

## Production Notes

**Know each function's buffering behavior before pointing it at a large or infinite stream.** The memory profile varies function by function, not library-wide. `ilen` consumes the whole iterator to count it; `seekable` without `maxlen` grows without bound; `divide` builds a list of the full input while `distribute` streams. Treat the docstring's memory note as part of the signature.

**Single-use iterator semantics bite here more than anywhere.** Because the library encourages passing raw iterators around, the classic Python footgun — feeding an already-exhausted generator into a second helper and silently getting nothing — shows up frequently. Materialize or `tee` deliberately.

**Pure Python is slower than the stdlib's C.** On Python 3.10+/3.12+, prefer `itertools.pairwise` and `itertools.batched` in hot loops over the more-itertools equivalents; the C versions are meaningfully faster per item. more-itertools remains the right choice where no stdlib equivalent exists, or where you need its extra options (e.g., `chunked(strict=True)`).

**Major versions remove deprecated names.** The project deprecates and then deletes at major bumps, roughly annually in recent years. The canonical incident: version 6.0.0 (February 2019) dropped Python 2, and because pytest depended on more-itertools at the time, Python 2 CI pipelines across the ecosystem broke until pins landed[^4]. The lesson generalizes — this package is so widely present transitively that its major bumps ripple; pin an upper bound if you rely on deprecated names.

**Namespace drift vs stdlib.** Functions that graduated to `itertools` may differ in signature or edge-case behavior from their more-itertools ancestors. When both exist, check which one your code actually imports before assuming behavior.

## When to Use / When Not

**Use when:**
- You are about to hand-roll chunking, windowing, run-length, split-on-condition, or "exactly one" logic — a tested, named function almost certainly already exists here.
- You want the stdlib `itertools` docs recipes as real imports instead of copy-paste.
- You need stateful iteration helpers (`peekable`, `seekable`, `bucket`) without writing the caching yourself.
- Dependency weight matters: it is pure Python with zero dependencies.

**Avoid when:**
- The stdlib already covers it (`itertools.batched`, `itertools.pairwise`, `functools.reduce`) and the loop is hot — the C implementations win.
- You want a functional-programming *style* (currying, composition pipelines) rather than iterator utilities — that's toolz/funcy territory.
- You are streaming unbounded data and cannot audit per-function buffering behavior.
- A two-line generator expression is honest enough; importing a dependency to avoid it is negative-sum.

## Alternatives

- pytoolz/toolz — use instead when you want a functional standard library (curried functions, dict/function utilities), not just iterator helpers; cytoolz adds a C-accelerated build.
- Suor/funcy — use instead for a smaller grab-bag of practical helpers spanning sequences, dicts, and decorators with a terser style.
- python/cpython — plain `itertools` + a copied recipe is the zero-dependency answer when you need only one or two functions.
- EntilZha/PyFunctional — use instead when you prefer chained, Spark-like collection pipelines over free functions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012-04 | Initial release by Erik Rose. |
| 6.0.0 | 2019-02 | Dropped Python 2; broke Python 2 CI ecosystem-wide via pytest's dependency[^4]. |
| 8.0.0 | 2019-11 | Python 3 only line consolidates; API grows toward 100+ functions. |
| 9.0.0 | 2022 | Deprecated-function removals; Python 3.7+. |
| 10.0.0 | 2023 | Dropped Python 3.7; further removals of long-deprecated names. |
| 10.x | 2023–2026 | Steady additions (constrained_batches, interleave_evenly, concurrency helpers); actively pushed as of 2026-07[^1]. |

## References

[^1]: more-itertools documentation and changelog. https://more-itertools.readthedocs.io/en/stable/
[^2]: CPython documentation, "itertools — Recipes". https://docs.python.org/3/library/itertools.html#itertools-recipes
[^3]: Python 3.12 release notes, `itertools.batched`. https://docs.python.org/3/whatsnew/3.12.html
[^4]: more-itertools version history (6.0.0: Python 2 support dropped). https://more-itertools.readthedocs.io/en/stable/versions.html

## Tags

python, iterators, itertools, utility-library, functional-programming, zero-dependencies, data-processing, stdlib-extension, lazy-evaluation
