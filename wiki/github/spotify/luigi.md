# spotify/luigi

> A Python library for building batch pipelines as dependency graphs, where task completion is defined by the existence of output targets rather than by a scheduler's run log.

[GitHub repo](https://github.com/spotify/luigi) ·
[Documentation](https://luigi.readthedocs.io/en/stable/) ·
[License: Apache-2.0](https://github.com/spotify/luigi/blob/master/LICENSE)

## Overview

Luigi is a Python module for stitching long-running batch jobs into pipelines. It was built at Spotify — mainly by Erik Bernhardsson and Elias Freider — and open sourced in late 2012[^1]. Its stated purpose is to handle the "plumbing" around batch processes: dependency resolution, failure handling, command-line integration, and a web visualiser of the task graph. It ships with a contrib toolbox for Hadoop, Hive, Pig, Spark, and file-system abstractions for HDFS, S3, and GCS, but it is not itself a compute engine — it orchestrates tasks that run elsewhere[^2].

The defining design choice is that **a task is "complete" when its output Target exists**, not when a scheduler records a successful run. A Task declares `requires()` (upstream tasks), `output()` (one or more Targets — usually files or table partitions), and `run()` (the work). Before running, Luigi checks whether the outputs already exist and skips tasks whose outputs are present. This makes pipelines idempotent and re-runnable by construction: a failed pipeline can be re-invoked and only the incomplete tail re-executes. It also means correctness hinges on Targets being written atomically, so a crash never leaves a half-written output that reads as "done."

Luigi's second defining trait is what it deliberately does *not* do. It has no built-in cron, no time-based triggering, and (in its classic model) no rich UI for authoring or backfilling. You trigger pipelines yourself — typically from cron, a CI job, or an external scheduler — and Luigi resolves and runs the dependency tail. This minimalism is why it lost mindshare to Airflow and the newer orchestrators, and also why teams who want a small, legible library rather than a platform still reach for it.

## Getting Started

```bash
pip install luigi          # core
pip install luigi[toml]    # optional TOML config support
```

```python
# pipeline.py
import luigi

class GenerateWords(luigi.Task):
    def output(self):
        return luigi.LocalTarget("words.txt")

    def run(self):
        # atomic: temp file swapped into place on close
        with self.output().open("w") as f:
            f.write("apple\nbanana\napple\n")

class CountLetters(luigi.Task):
    def requires(self):
        return GenerateWords()

    def output(self):
        return luigi.LocalTarget("counts.txt")

    def run(self):
        with self.input().open("r") as fin:
            words = fin.read().split()
        with self.output().open("w") as fout:
            for w in words:
                fout.write(f"{w}\t{len(w)}\n")

if __name__ == "__main__":
    luigi.build([CountLetters()], local_scheduler=True)
```

Run it, or invoke from the command line with `--local-scheduler`:

```bash
python -m luigi --module pipeline CountLetters --local-scheduler
```

## Architecture / How It Works

Three abstractions carry the whole model:

- **Task** — a unit of work. Identity is derived from the class name plus its Parameters, so `MyTask(date=2026-07-15)` and `MyTask(date=2026-07-14)` are distinct nodes in the graph. Parameters are typed (`luigi.DateParameter`, `IntParameter`, etc.) and double as CLI flags.
- **Target** — a checkable output: `LocalTarget`, `S3Target`, `HdfsTarget`, database/table targets, and others in contrib. `target.exists()` is the completeness oracle. `open("w")` on file targets writes to a temp path and renames on close for atomicity.
- **Parameter** — typed, hashable inputs that make tasks reproducible and addressable.

**Scheduling.** There are two schedulers. The *local scheduler* (`--local-scheduler`) resolves and runs the graph in-process — good for development and single-box runs. The *central scheduler* (`luigid`, a daemon) coordinates multiple workers, prevents two workers from running the same task concurrently via a task-ID lock, and serves the web visualiser. The central scheduler holds state **in memory by default** and is not a durable database of runs — it is a coordinator, not a system of record.

**Dynamic dependencies.** Beyond the static `requires()`, a `run()` method can `yield` additional Task instances at execution time. Luigi pauses the task, runs the yielded dependencies, then resumes. This supports fan-out where the dependency set is only known after some computation.

**Execution model.** Workers pull runnable tasks (those whose dependencies are complete) from the scheduler and execute `run()` in the worker process. There is no isolation boundary by default — a task runs as Python in the worker, so heavy work is usually delegated to Hadoop/Spark/SQL via contrib wrappers rather than done in-process.

**Configuration** is file-based: `luigi.cfg` / `client.cfg` (INI) or TOML with the `[toml]` extra. Parameter defaults, the scheduler host, retry policy, and per-task settings live there and are overridable via environment and CLI.

## Production Notes

- **No triggering means you own the cron.** Luigi will not start a pipeline on a schedule. In production you wrap `luigi` invocations in cron/systemd timers/CI. Retries of a *whole pipeline* are cheap because completed targets are skipped, but you still need external logic to decide *when* to fire and how to alert on stuck runs.
- **The central scheduler is not durable.** Restarting `luigid` loses in-memory run history unless you have persisted it externally. Treat it as a live coordinator; push audit history (which partitions were produced when) into your own outputs, not into the scheduler.
- **Atomicity is a contract you must uphold.** The whole "output exists ⇒ done" model breaks if a task writes a partial output that still passes `exists()`. Use the provided atomic file targets; for custom targets, write to a temp location and move into place. A non-atomic `output()` is the classic Luigi footgun — pipelines silently skip re-running because a truncated file "exists."
- **Parameter-based identity is a sharp edge.** Change a Parameter's default or type and you change task identity, which can invalidate or orphan previously-completed outputs. Date algebra in parameters (rolling windows, backfills) is powerful but makes it easy to accidentally request thousands of tasks.
- **Scaling is worker-bound, not engine-bound.** Luigi does not distribute compute; it distributes *coordination*. Throughput of the pipeline is limited by how many workers you run and how fast the underlying jobs (Hadoop/Spark/SQL) complete. Very wide graphs put pressure on the single central scheduler.
- **Maintenance posture.** The project is still maintained by Spotify's Data team and receives commits and releases (recent Python versions through 3.14 are listed as tested), but it is mature and slow-moving rather than actively expanding. New greenfield orchestration projects overwhelmingly start on Airflow, Prefect, or Dagster; Luigi's active base is largely existing pipelines and teams that value its small surface area.

## When to Use / When Not

**Use when:**
- Your pipeline is fundamentally "produce files/partitions in dependency order," and idempotent re-runs matter more than a scheduling UI.
- You want a small, legible Python library you can read end to end, not a platform to operate.
- You already trigger jobs from cron/CI and just need dependency resolution + skip-if-output-exists.
- You are orchestrating Hadoop/Hive/Spark/SQL jobs and want thin, well-worn wrappers.

**Avoid when:**
- You need built-in scheduling, backfills, a rich authoring/monitoring UI, and a durable run history out of the box.
- You want dynamic, data-aware DAGs with first-class retries, SLAs, and alerting managed by the tool.
- You are building streaming or low-latency pipelines — Luigi is batch-only by design.
- Your team wants asset-/data-lineage as the primary abstraction rather than tasks and targets.

## Alternatives

- apache/airflow — use instead when you need built-in scheduling, backfills, a mature UI, and a large operator ecosystem; it is the default choice for new batch orchestration.
- PrefectHQ/prefect — use instead when you want Pythonic dynamic flows, managed retries, and a hosted control plane without writing your own triggering.
- dagster-io/dagster — use instead when data assets and lineage (not tasks) are the right primitive and you want typing/testing baked in.
- argoproj/argo-workflows — use instead when your jobs are containers on Kubernetes and you want the scheduler to be K8s-native.
- apache/oozie — use instead only in a legacy Hadoop/XML-configured environment where it is already the standard.

## History

| Version | Date | Notes |
|---------|------|-------|
| open source | 2012-09 | Released by Spotify on GitHub[^1]. |
| 1.x | 2015 | Arash Rouhani chief maintainer 2015–2019; broad contrib growth[^1]. |
| 2.0 | 2015 | Python 3 support alongside Python 2. |
| 3.0 | 2020 | Python 2 support dropped; Python 3 only. |
| 3.x (current) | 2026 | Maintained by Spotify Data team; Python 3.10–3.14 tested per README[^2]. |

## References

[^1]: Luigi README, "Authors" and "Background" — Spotify, Erik Bernhardsson and Elias Freider; open sourced late 2012, Arash Rouhani chief maintainer 2015–2019. https://github.com/spotify/luigi
[^2]: Luigi documentation (Read the Docs). Contrib modules, Target/Task model, and tested Python versions. https://luigi.readthedocs.io/en/stable/

## Tags

python, data-engineering, workflow-orchestration, batch-processing, etl, dag, pipelines, hadoop, spotify, dependency-resolution
