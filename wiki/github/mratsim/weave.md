# mratsim/weave

> A message-passing multithreading runtime for Nim — "work-requesting" instead
> of shared-memory work-stealing, engineered for ultra-low per-task overhead.

[GitHub repo](https://github.com/mratsim/weave) ·
[License: MIT OR Apache-2.0](https://github.com/mratsim/weave#license)

## Overview

Weave (codenamed "Project Picasso") is a task-parallel and data-parallel
runtime for the Nim programming language, written by Mamy Ratsimbazafy
(mratsim)[^1]. Its research pedigree is the distinguishing feature: instead of
the classic Cilk/TBB design where idle workers steal tasks from other workers'
shared-memory deques, Weave workers *send steal requests over channels* and
victims push work back — a design called work-requesting, derived from Andreas
Prell's tasking-2.0 research runtime[^2]. All inter-worker synchronization is
reduced to SPSC and MPSC channels, which shrinks the lock-free surface area.
The README claims 3–10x lower overhead than Intel TBB and GCC OpenMP on
overhead-bound benchmarks (billions of subtasks), with benchmarks drawn from
recursive tree algorithms, linear algebra, and HPC workloads[^1].

The defining tension is research quality versus production maturity. Weave
self-labels as *experimental*: only one of its two complex synchronization
primitives was formally verified deadlock-free, and the worker state machines
were not verified at all[^1]. The repository is dormant — the last release
(v0.4.10) and last push date to June 2024[^4] — and the author's subsequent
production work went into simpler designs: status-im/nim-taskpools (built to
be auditable for the Nimbus Ethereum client)[^3] and the internal threadpool
of mratsim/constantine. At 585 stars it remains the Nim ecosystem's most
complete parallel-runtime experiment, and a readable reference implementation
of message-passing scheduling.

## Getting Started

```bash
nimble install weave   # requires Nim >= 1.2.0
```

```nim
import weave

proc fib(n: int): int =
  if n < 2:
    return n
  let x = spawn fib(n-1)   # returns a Flowvar (future)
  let y = fib(n-2)
  result = sync(x) + y     # await the Flowvar

proc main() =
  init(Weave)              # thread calling init becomes the root thread
  let f = fib(20)
  exit(Weave)
  echo f

main()
```

Data parallelism uses `parallelFor` with explicit `captures: {...}` blocks;
loops nest freely and are load-balanced by adaptive splitting.

## Architecture / How It Works

- **Work-requesting scheduler.** Idle workers never touch other workers'
  queues; they send a steal request through a channel and a busy victim ships
  tasks back. Load balancing is thus done by the *busy* workers,
  cooperatively, at task boundaries or explicit `loadBalance(Weave)` calls[^1].
- **Channels only.** Synchronization is limited to SPSC channels (task/result
  handoff) and MPSC channels (steal requests), avoiding concurrent deques and
  their memory-reclamation problems[^1].
- **Workers as state machines.** One worker thread per logical core (capped by
  `WEAVE_NUM_THREADS`); the thread calling `init(Weave)` becomes the
  privileged root thread that alone may call the global barrier `syncRoot`.
- **Futures (`Flowvar`) and scopes.** `spawn`/`sync` mirror async/await;
  `syncScope` is a composable structured-concurrency barrier over all
  descendant tasks. Lazy Flowvar allocation (experimental) elides allocation
  for tasks resolved on the spawning thread.
- **Dataflow parallelism.** `FlowEvent` handles delay tasks and individual
  loop-iteration ranges until data dependencies are met — producer-consumer
  graphs rather than pure fork-join[^1].
- **Memory pool.** A custom arena allocator (16 KB base, tunable via
  `-d:WV_MemArenaSize=...`) serves task and channel allocations[^1].
- **Background service (experimental).** `runInBackground` turns Weave into a
  FIFO job executor: foreign threads `submit` jobs and `waitFor` results.

## Production Notes

- **Cooperative scheduling is the big footgun.** Because busy workers perform
  the load balancing, a thread that blocks, sleeps, or grinds through a long
  non-Weave computation starves the whole runtime — other workers spin on
  unanswered steal requests. You must sprinkle `loadBalance(Weave)` inside
  long kernels and call `syncRoot(Weave)` before blocking the root thread[^1].
- **GC-managed types don't cross task boundaries.** The documented pattern
  casts `seq` buffers to `ptr UncheckedArray[T]` before capture; captures are
  plain-data copies. Accidentally capturing a `seq` or `ref` is an easy
  mistake, and the design predates Nim's ARC/ORC default memory model.
- **Experimental, and verified only in part.** Per the README's own
  disclaimer: one of two complex sync primitives verified deadlock-free, no
  data-race-detection pass, unverified worker state machines[^1].
- **Dormancy risk.** No commits since June 2024[^4]; 44 open issues. For
  correctness-critical Nim services the author's own later choice was the
  simpler, auditable status-im/nim-taskpools[^3] — a telling signal about
  where Weave sits on the risk curve.
- **Platform edges.** Windows 32-bit requires MSVC (MinGW lacks
  `EnterSynchronizationBarrier`); `syncScope` miscompiles inside for loops on
  the C++ backend due to an upstream Nim bug; macOS pthreads lack barriers and
  affinity, so those are emulated or unused[^1].
- **Backoff.** Idle workers sleep by default; disabling it
  (`-d:WV_Backoff=off`) means 100% CPU spin on idle workers.

## When to Use / When Not

**Use when:**
- You are writing compute-bound Nim (numerics, simulation, HPC-style kernels)
  with fine-grained or irregular tasks where scheduler overhead dominates.
- You need dataflow/pipeline dependencies (`FlowEvent`), not plain fork-join.
- You are studying scheduler design — the code and its research trail are the
  best documented in the Nim ecosystem.

**Avoid when:**
- You need a maintained, production-audited pool: use status-im/nim-taskpools.
- Your workload mixes I/O or blocking calls with compute — Weave's cooperative
  model degrades badly around blocked threads.
- You want to pass GC-managed Nim types between tasks without pointer casts.
- You are not on Nim; this runtime is language-specific by construction.

## Alternatives

- status-im/nim-taskpools — same author, deliberately minimal and auditable;
  use it for production Nim services where correctness review beats overhead.
- mratsim/constantine — its internal threadpool is the maintained descendant
  combining Weave's research with taskpools' simplicity.
- oneapi-src/oneTBB — the C/C++ industry-standard work-stealing runtime Weave
  benchmarks against; use it outside Nim.
- taskflow/taskflow — C++ task-graph parallelism with dataflow dependencies
  comparable to Weave's FlowEvent model.
- rayon-rs/rayon — the Rust analog for data parallelism; use it when language
  choice is open and you want a large production track record.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 "Arabesques" | 2019-12-15 | First release, ~5 months after repo creation[^4]. |
| v0.2.0 "Overture" | 2019-12-22 | Rapid follow-up release. |
| v0.3.0 "Beam me up!" | 2020-01-01 | |
| v0.4.0 "Bespoke" | 2020-04-04 | Last minor version; the v0.4.x line spans the rest of the project's life. |
| v0.4.10 | 2024-06-29 | Final release to date; coincides with the last push to the repo[^4]. |

## References

[^1]: Weave README — design, benchmarks, disclaimers, platform notes. https://github.com/mratsim/weave#readme
[^2]: Andreas Prell, tasking-2.0 — the work-requesting runtime Weave's scheduler derives from. https://github.com/aprell/tasking-2.0
[^3]: status-im/nim-taskpools — "lightweight, auditable" threadpool built for the Nimbus client by the same author. https://github.com/status-im/nim-taskpools
[^4]: Weave releases. https://github.com/mratsim/weave/releases

## Tags

nim, multithreading, work-stealing, message-passing, task-parallelism, data-parallelism, scheduler, runtime, hpc, fork-join
