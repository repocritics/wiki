# treeverse/dvc

> Git-style version control for datasets, ML models, and reproducible pipelines — metadata in Git, bytes in your own storage.

[GitHub repo](https://github.com/iterative/dvc) ·
[Official website](https://dvc.org) ·
[License: Apache-2.0](https://github.com/iterative/dvc/blob/main/LICENSE)

## Overview

DVC (Data Version Control) is a command-line tool, built by Iterative, that brings Git-like workflows to the large files Git itself handles badly: training datasets, model weights, and intermediate ML artifacts[^1]. Instead of committing multi-gigabyte blobs, you commit small text pointer files (`*.dvc`, `dvc.lock`) that Git tracks normally, while the actual bytes live in a content-addressable cache and are synced to remote storage you own (S3, GCS, Azure Blob, SSH, HDFS, local, and others).

The tool has three loosely-coupled layers that ship together but are used independently: **data versioning** (`dvc add` / `dvc push` / `dvc pull`), **pipelines** (a `dvc.yaml` DAG of stages that reruns only what changed, `dvc repro`), and **experiment tracking** (`dvc exp run`, which records parameters, metrics, and plots as lightweight Git objects with no server)[^2]. Teams often adopt only the first layer and treat DVC as "Git-LFS without a hosted LFS server."

The defining tension is that DVC is deliberately Git-shaped in a domain that is not. Datasets do not diff, merge, or branch the way source code does, and the pointer-file indirection means every data operation is two-phase: Git tracks the pointer, DVC moves the bytes. This buys reproducibility and storage-agnosticism at the cost of an extra mental model, a cache that can silently double disk usage, and status/checkout operations that get slow on directories with very large file counts.

## Getting Started

```bash
pip install "dvc[s3]"   # extras: s3, gs, azure, ssh, oss, gdrive, or 'all'
# also available via: brew install dvc, conda install -c conda-forge dvc, snap
```

```bash
git init && dvc init            # dvc init adds .dvc/ alongside .git/

dvc add data/images            # hashes the dir, writes data/images.dvc pointer
git add data/images.dvc .gitignore
git commit -m "track image dataset"

dvc remote add -d storage s3://my-bucket/dvcstore
dvc push                        # upload cached bytes to the remote
```

```yaml
# dvc.yaml — a two-stage pipeline; `dvc repro` reruns only changed stages
stages:
  featurize:
    cmd: python featurize.py
    deps: [data/images, featurize.py]
    outs: [data/features]
  train:
    cmd: python train.py
    deps: [data/features, train.py]
    params: [lr, epochs]
    outs: [model.pkl]
    metrics: [metrics.json]
```

## Architecture / How It Works

**Pointer files and the cache.** `dvc add data/images` computes a hash of the file or directory and writes a small YAML pointer (`data/images.dvc`) containing that hash. The real bytes are moved into `.dvc/cache`, a content-addressable store keyed by hash. The working-tree copy is then relinked from the cache. Directories are tracked as a single hash over a JSON listing of their contents, so a folder of a million files is one pointer, not a million.

**Linking strategy is the central performance knob.** To avoid storing every file twice (once in cache, once in the workspace), DVC links workspace files back to the cache using reflinks, hardlinks, or symlinks — falling back to a full **copy** when the filesystem does not support the chosen method. The default is conservative (copy on filesystems without reflink support), which is safe but doubles disk usage. `dvc config cache.type reflink,hardlink,symlink` selects a preference order; hardlink/symlink modes make cached files read-only to prevent corrupting shared cache entries[^3].

**Remotes.** A remote is just a URL to object storage. `dvc push` uploads cache objects, `dvc pull` fetches them, `dvc gc` prunes unreferenced ones. There is no DVC server — the remote is dumb storage, and access control is whatever the storage backend provides.

**Pipelines.** `dvc.yaml` declares stages with `deps`, `outs`, `params`, and `metrics`. `dvc repro` walks the DAG, hashes dependencies, and skips stages whose inputs are unchanged, recording results in `dvc.lock`. This is conceptually a Makefile for ML with content-hashing instead of mtimes.

**Experiments.** `dvc exp run` executes the pipeline and stores the result as a hidden Git commit under custom refs (`refs/exps/...`) rather than polluting your branch history. `dvc exp show` renders a table of params/metrics across experiments; `dvc exp apply` promotes one into your workspace. Because experiments are Git objects, they push/pull through normal Git remotes.

## Production Notes

- **Cache disk blowup.** On ext4 and most non-CoW filesystems without a configured link type, DVC copies data, so a 50 GB dataset consumes ~100 GB (cache + workspace). Configure `cache.type` to `reflink` (APFS, Btrfs, XFS) or `hardlink`/`symlink` early. With hardlink/symlink, workspace files are read-only — editing them in place is a common surprise.
- **Slow on many small files.** Hashing, `dvc status`, and `dvc checkout` scale with file count, not just size. Directories with hundreds of thousands of files are a known pain point; prefer packing into archives or fewer large files when possible.
- **`dvc.lock` merge conflicts.** Because the lockfile records hashes of every output, parallel branches that both rerun a pipeline will conflict on `dvc.lock`. Resolving means picking one side and re-running `dvc repro`.
- **Hashing cost.** File hashes are computed on add/status/checkout. For very large individual files this is CPU/IO-bound; there is no way around reading the bytes.
- **Windows.** Symlink creation historically requires elevated privileges or Developer Mode, pushing Windows users toward `copy` or `hardlink` cache types.
- **`dvc gc` is destructive.** Garbage collection removes cache objects not referenced by the current scope; run with the right `--all-branches`/`--all-tags`/`--all-commits` flags or you can delete data still referenced by other branches. On remotes, `dvc gc -c` deletes from cloud storage.
- **The 3.0 cache-format migration.** DVC 3.0 (2023) changed the on-disk cache layout and hashing details; mixing old and new client versions across a shared cache/remote can require a one-time migration[^4]. Pin the DVC version in CI.
- **Iterative's product churn.** DVC the OSS tool is stable and maintained, but surrounding Iterative products have shifted (Studio, MLEM, GTO, and the DataChain pivot). Treat DVC core as the durable piece and adjacent tooling as more volatile.

## When to Use / When Not

**Use when:**
- You need dataset/model versions tied to Git commits but cannot commit the bytes themselves.
- You want storage-agnostic data sync (bring your own S3/GCS/Azure) with no vendor-hosted data plane.
- You want reproducible, cache-aware ML pipelines that rerun only changed stages.
- You want lightweight, serverless experiment tracking that lives in Git.

**Avoid when:**
- Your data is many millions of tiny files or streams continuously — DVC's per-file hashing and pointer model will fight you; a data lake / table format fits better.
- You want a hosted, collaborative UI with governance and lineage out of the box (managed platforms cover that).
- Your team wants pure experiment tracking without the Git-data model — a dedicated tracker is lighter.
- You need concurrent multi-writer versioning of a shared dataset; DVC's Git-branch semantics assume code-like, low-contention edits.

## Alternatives

- git-lfs/git-lfs — simpler large-file storage bolted onto Git; use when you just need big binaries versioned and don't need pipelines or experiments.
- pachyderm/pachyderm — Kubernetes-native data pipelines with data-driven, automatic versioning; use when you need containerized pipeline orchestration at scale rather than a local CLI.
- treeverse/lakeFS — Git-like branching over object storage at data-lake scale; use when versioning happens in the storage layer over petabytes, not in a local cache.
- mlflow/mlflow — experiment tracking + model registry; use (often alongside DVC) when tracking runs and serving models matters more than versioning the data bytes.
- activeloopai/deeplake — a versioned tensor/dataset store with streaming; use when you want dataset versioning integrated with a query/loader layer for training.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017-05 | Initial open-source release; data versioning via `.dvc` pointer files[^1]. |
| 1.0 | 2020-06 | Multi-stage `dvc.yaml` pipelines and `dvc.lock`[^2]. |
| 2.0 | 2021-04 | Experiment management (`dvc exp`), lightweight Git-based runs. |
| 3.0 | 2023-06 | Cache-format and hashing changes; tighter DVCLive integration[^4]. |

## References

[^1]: DVC documentation, "What is DVC?". https://dvc.org/doc/user-guide/what-is-dvc
[^2]: DVC documentation, "Data Pipelines". https://dvc.org/doc/user-guide/pipelines
[^3]: DVC documentation, "Large Dataset Optimization / cache types". https://dvc.org/doc/user-guide/data-management/large-dataset-optimization
[^4]: DVC 3.0 release notes. https://dvc.org/blog/dvc-3-0-ml-experiments-data-and-model-versioning
[^5]: Barrak, Eghan, Adams, "On the Co-evolution of ML Pipelines and Source Code — Empirical Study of DVC Projects," SANER 2021. https://mcis.cs.queensu.ca/publications/2021/saner.pdf

## Tags

python, data-version-control, mlops, machine-learning, reproducibility, data-engineering, experiment-tracking, cli, pipelines, model-versioning
