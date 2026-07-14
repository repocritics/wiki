# huggingface/datasets

> Apache Arrow-backed loader and preprocessing layer for ML datasets — the `load_dataset()` on-ramp to the Hugging Face Hub.

[GitHub repo](https://github.com/huggingface/datasets) ·
[Official website](https://huggingface.co/docs/datasets) ·
[License: Apache-2.0](https://github.com/huggingface/datasets/blob/main/LICENSE)

## Overview

`datasets` is the Python library that most ML training pipelines use to pull a
corpus and get it into a trainable shape. It began life as `huggingface/nlp` and
was renamed at the 1.0 release in September 2020[^1]. Its scope is deliberately
narrow: fetch a dataset (from the Hub or local files), give it a typed schema,
and let you `.map()` transforms over it — while memory-mapping the bytes through
Apache Arrow so a dataset larger than RAM behaves like an in-memory list.

The library's defining bet is Arrow. Every `Dataset` is a memory-mapped Arrow
table, which is what lets a 200 GB corpus load in seconds and share pages across
worker processes without copying. The same bet is the source of most of its
friction: Arrow is columnar and immutable, so per-row Python access, mutation,
and shuffling all cut against the grain and go through a reconstruction path.

The second defining tension is historical. For its first four years the primary
way to add a dataset was a *loading script* — arbitrary Python committed to the
Hub that `load_dataset()` executed on your machine (`trust_remote_code=True`).
That model was flexible and a genuine remote-code-execution surface. Version 3.0
(2024) removed script-based loading from the default path in favor of data-only
formats (Parquet, CSV, JSON, WebDataset)[^2]. The cleanup improved security and
reproducibility but broke every dataset that had depended on a custom script.

## Getting Started

```bash
pip install datasets
# extras: datasets[audio]  datasets[vision]  datasets[torch,jax]
```

```python
from datasets import load_dataset

# Streams into a memory-mapped Arrow table under ~/.cache/huggingface/datasets
squad = load_dataset("rajpurkar/squad")
print(squad["train"][0])          # random access by row index

# .map() is the core transform; results are cached on disk by content hash
tokenized = squad.map(
    lambda ex: {"length": len(ex["context"])},
    batched=True,
    num_proc=4,                    # fork workers, share the mmap
)

# Hand off to a PyTorch DataLoader without copying tensors out of Arrow
tokenized = tokenized.with_format("torch")
```

```python
# Datasets bigger than disk: iterate without downloading
stream = load_dataset("timm/imagenet-1k-wds", streaming=True)
for example in stream["train"]:
    print(example["image"]); break
```

## Architecture / How It Works

There are two dataset types, and the distinction governs everything:

1. **`Dataset`** — a memory-mapped Arrow table. Supports random access
   (`ds[42]`), slicing, `select()`, and `sort()`. Backed by files on disk;
   indexing is O(1) and the process's resident memory stays flat regardless of
   corpus size.
2. **`IterableDataset`** — a lazy pipeline with no index. This is what
   `streaming=True` returns. You can only iterate forward; `map`/`filter` are
   applied on-the-fly per element, and shuffling is a bounded-buffer
   approximation, not a true permutation.

Both are wrapped in `DatasetDict` (a plain dict of splits) when a dataset has
`train`/`test`/`validation`.

A **`Features`** schema types every column — `Value("int64")`, `ClassLabel`,
`Sequence`, `Image`, `Audio`, `Json`. Media columns store a path or bytes plus a
decoder, so an `Image` column doesn't materialize pixels until you read the row.

**Caching is fingerprint-based.** Every `.map()`/`.filter()` computes a hash of
the previous fingerprint plus a Dill pickle of your transform function and its
closure. Matching hash → the result Arrow file is reused; no match → recompute.
This is what makes reruns instant, and also the source of the library's most
notorious footguns (below).

`num_proc` forks worker processes that each mmap the shared Arrow file, so
multiprocessing avoids copying the data — but your `map` function and its
arguments must pickle cleanly to cross the fork. FAISS and Elasticsearch indexes
can be attached to a column for `get_nearest_examples()` similarity search.

## Production Notes

**The cache is the thing that will bite you.** Two failure modes are common and
opposite: (a) `.map()` *over*-caches — you edit your transform, the fingerprint
happens to collide or your function isn't re-hashed, and you silently get stale
results; pass `load_from_cache_file=False` when iterating on preprocessing. (b)
`.map()` *under*-caches — a lambda, a local closure, or a non-picklable object
(an open file, a CUDA tensor, a `functools.partial` over an unhashable) makes
Dill produce a random fingerprint every run, so the expensive map recomputes
each time and fills disk with duplicate Arrow files.

**Disk, not RAM, is the constraint.** `~/.cache/huggingface/datasets` accumulates
the raw download *and* one Arrow file per cached transform. On shared training
boxes this grows without bound; set `HF_DATASETS_CACHE` to a scratch volume and
prune it, or use `keep_in_memory=True` for small datasets.

**Random access after shuffle is slow.** `Dataset.shuffle()` doesn't reorder the
Arrow bytes; it builds an indices mapping, so subsequent row reads jump around
the mmap and lose sequential-read locality. For training throughput, prefer
`.flatten_indices()` after a shuffle, or stream with `IterableDataset` and a
shuffle buffer.

**Streaming trades exactness for scale.** `IterableDataset` shuffling only mixes
within a buffer (`buffer_size`), so it is a weak shuffle; splitting across
`DataLoader` workers requires `split_dataset_by_node` / correct `num_workers`
handling or you get duplicated samples.

**Upgrade pain is real.** The 2.x→3.0 removal of loading scripts is the big one:
datasets that shipped a `.py` loader stopped resolving under the default path,
and audio decoding moved to `torchcodec` (replacing `soundfile`/`librosa`),
which pulls in FFmpeg and changed the shape of decoded audio for some pipelines.
Pin `datasets` in training environments and pin the dataset `revision` for
reproducibility, as the README itself advises.

## When to Use / When Not

**Use when:**
- You're pulling public corpora from the Hugging Face Hub — this is the native path.
- Your data is larger than RAM and you want memory-mapping for free.
- You want cached, reproducible preprocessing with `.map()` across train runs.
- You need one interface that yields NumPy, Pandas, Polars, PyTorch, TF, or JAX.

**Avoid when:**
- Your data is small tabular and you just want a dataframe — Polars or Pandas is simpler.
- You need mutable, row-oriented, transactional data — Arrow is immutable and columnar.
- You're doing petabyte-scale distributed training with cloud-native sharding — a purpose-built streaming loader fits better.
- You want zero external cache state — the on-disk cache is load-bearing, not optional.

## Alternatives

- webdataset/webdataset — use instead when you want tar-shard streaming for large-scale distributed training without an Arrow cache.
- mosaicml/streaming — use instead when training from cloud object storage (S3/GCS) with deterministic resumable sharding is the priority.
- pola-rs/polars — use instead when the data is tabular and you want a fast dataframe, not an ML dataloader.
- pytorch/data (torchdata) — use instead when you want composable DataPipes wired directly into the PyTorch `DataLoader` ecosystem.
- activeloopai/deeplake — use instead when you want a versioned tensor database with its own storage format and querying.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-03 | Started as `huggingface/nlp`. |
| 1.0 | 2020-09 | Renamed to `datasets`; Arrow-backed loader, loading scripts[^1]. |
| 2.0 | 2022-03 | Streaming maturation, Hub-first dataset resolution. |
| 3.0 | 2024-09 | Removed script-based loading from default path; audio via torchcodec[^2]. |

## References

[^1]: Lhoest et al., "Datasets: A Community Library for Natural Language Processing," EMNLP 2021 System Demonstrations. https://aclanthology.org/2021.emnlp-demo.21 (arXiv:2109.02846)
[^2]: Hugging Face Datasets documentation — loading and dataset formats. https://huggingface.co/docs/datasets/loading
[^3]: Repository metadata via GitHub API, retrieved 2026-07-14. https://github.com/huggingface/datasets

## Tags

python, machine-learning, datasets, apache-arrow, data-loading, nlp, huggingface, deep-learning, streaming, preprocessing
