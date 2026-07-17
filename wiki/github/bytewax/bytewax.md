# bytewax/bytewax

> Python stream processing with a Rust dataflow engine — Timely Dataflow with a Pythonic operator API.

[GitHub repo](https://github.com/bytewax/bytewax) ·
[Official website](https://docs.bytewax.io/) ·
[License: Apache-2.0](https://github.com/bytewax/bytewax/blob/main/LICENSE)

## Overview

Bytewax is a stateful stream-processing framework: you write dataflows in Python, and they execute on a Rust engine built on Timely Dataflow, the same academic dataflow runtime (Frank McSherry et al.) that underlies Materialize[^1]. The pitch is to give Python data teams Flink/Kafka-Streams-style stateful streaming — windowing, joins, recoverable state — without leaving the Python ecosystem or writing Java/Scala. Operators (`map`, `filter`, `fold_window`, `join`, `stateful_map`) are composed into a `Dataflow` graph and run with `python -m bytewax.run`.

The defining tension is the language boundary. The engine is fast native Rust, but every user operator is a Python callable, so throughput is gated by Python execution and serialization across the FFI edge. Bytewax sidesteps the GIL by running multiple OS-process workers rather than threads — each worker is its own Python interpreter — which is how it scales, but it also means all the usual multiprocess concerns (no shared memory, per-worker state partitioning by key) apply.

The larger caveat is governance. As of May 2025 the company behind Bytewax wound down commercial operations and the original core team stepped back; the project is now explicitly community-maintained, with the maintainer pool being rebuilt[^2]. The last tagged release, 0.21.1, is from November 2024[^3]. It is open source under Apache-2.0 and the code still works, but anyone adopting it in 2026 is adopting a project without a funded team behind it — evaluate accordingly.

## Getting Started

```sh
pip install bytewax        # requires Python >=3.8; ships prebuilt Rust wheels
```

```python
# quickstart.py
from bytewax.dataflow import Dataflow
from bytewax import operators as op
from bytewax.testing import TestingSource

flow = Dataflow("quickstart")

inp = op.input("inp", flow, TestingSource([1, 2, 3, 4, 5]))
evens = op.filter("keep_even", inp, lambda x: x % 2 == 0)
scaled = op.map("times_10", evens, lambda x: x * 10)
op.inspect("out", scaled)
```

```sh
python -m bytewax.run quickstart.py        # single worker
python -m bytewax.run quickstart.py -w 2    # 2 worker threads in one process
```

The API shown here is the post-0.16 dataflow model. Older tutorials using `Dataflow()` with chained `.map()` methods target the pre-0.16 API and will not run on current versions (see History).

## Architecture / How It Works

Bytewax is a thin Python layer over a Timely Dataflow program. The `Dataflow` object you build in Python is compiled into a timely dataflow graph; the Rust runtime schedules operators, moves data between workers, and manages state.

- **Workers and parallelism.** A running dataflow is one or more workers. Within a process you get worker threads (`-w`); across machines you launch multiple processes (`-p` / `-a` addresses, or `waxctl` on Kubernetes) that connect over TCP. Because Python operators hold the GIL, real CPU parallelism comes from multiple processes, not threads. State is partitioned by key across workers via consistent routing, so a stateful operator's key space is sharded and each key lives on exactly one worker.
- **Stateful operators and recovery.** Stateful operators (`stateful_map`, `reduce`, window operators) keep per-key state. Bytewax implements recovery by periodically snapshotting state to a recovery store (SQLite partitions) so a restarted cluster can resume from the last epoch rather than replaying from the source[^4]. Recovery is opt-in and requires the source to be replayable (e.g. Kafka offsets) for correctness.
- **Windowing.** Event-time and processing-time windows (tumbling, sliding, session) with a clock/watermark abstraction. Event-time correctness depends on supplying a timestamp extractor and a watermark policy; late data handling is explicit.
- **Connectors.** Inputs and outputs are `Source`/`Sink` classes. Built-ins include Kafka (`bytewax.connectors.kafka`), stdio, and files; a partitioned connector API lets you write custom sources that participate in recovery. Additional connectors live in the community Module Hub rather than the core repo.

The coupling worth understanding: Bytewax's execution semantics, backpressure, and fault-tolerance model are Timely Dataflow's, not something Bytewax invented. That is a strength (a mature, formally-studied dataflow core) and a constraint (behavior and limits are inherited from timely, and debugging deep issues can mean reasoning about the Rust engine).

## Production Notes

- **Maintenance risk is the headline.** No release since 0.21.1 (Nov 2024) and a community-maintained status since May 2025[^2][^3]. The repo still sees commits, but there is no funded team, no committed release cadence, and no guarantee of timely security or dependency fixes. Pin versions and budget for maintaining a fork if this is load-bearing.
- **The API broke hard at 0.16.** The 0.16 rewrite (April 2023) replaced the chained-method dataflow API with the `operators as op` functional API and changed the recovery system. Upgrading across that boundary is a rewrite, not a bump. Much of the third-party tutorial content online predates it and no longer runs.
- **Scaling is multiprocess.** To use N cores you run N worker processes, each a separate Python interpreter loading your flow module. Startup cost, memory footprint, and any module-level side effects multiply by worker count. Shared state must go through the dataflow (keyed routing), not Python globals.
- **Recovery has sharp edges.** State snapshotting to SQLite recovery partitions must be configured, sized, and stored durably; the number of recovery partitions is fixed at flow creation and interacts with rescaling. Exactly-once end-to-end depends on both a replayable source and an idempotent/transactional sink — Bytewax alone does not make arbitrary sinks exactly-once.
- **Kafka specifics.** The Kafka connector is the most production-exercised path. Consumer group semantics, partition-to-worker assignment, and offset commit timing relative to recovery epochs all matter for delivery guarantees; read the Kafka connector guide before relying on defaults.
- **Serialization at the boundary.** Data crossing the Python/Rust edge and moving between workers is serialized. Large or deeply-nested Python objects per event are a throughput cost; keep per-event payloads lean.
- **`waxctl` and Platform.** `waxctl` (the deployment CLI) and the "Bytewax Platform" were commercial/company assets. With the company wound down, treat their long-term availability and support as uncertain and prefer self-managed Kubernetes/container deployment.

## When to Use / When Not

**Use when:**
- Your team is Python-first and needs stateful streaming (windowed aggregations, joins, online ML features) without adopting the JVM.
- You want a lightweight, single-`pip install` engine you can run locally and scale to Kubernetes with the same code.
- Your logic genuinely needs streaming state and recovery, not just a consumer loop.

**Avoid when:**
- You need a vendor-supported, actively-released platform with SLAs — the maintenance status makes this a real risk in 2026.
- Throughput per core is critical and your per-event work is heavy Python — the language boundary and GIL push you toward more processes or a JVM/native engine.
- Your problem is simple Kafka consume-transform-produce with no state — a plain consumer or a lighter library is less machinery.

## Alternatives

- quixio/quix-streams — Python-native Kafka stream processing, actively developed and commercially backed; the closest live alternative if the Bytewax maintenance risk is disqualifying.
- faust-streaming/faust — community fork of Robinhood's Faust; pure-Python streaming with a similar audience, though also maintenance-constrained.
- apache/flink — use when you need a mature, heavily-supported engine (via PyFlink) and can accept JVM operational weight.
- TimelyDataflow/timely-dataflow — use directly when you want the underlying Rust dataflow engine and are willing to write Rust for maximum performance.
- apache/beam — use when you want a portable pipeline API across multiple runners (Dataflow, Flink, Spark) rather than a single engine.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.16.0 | 2023-04-19 | Major rewrite: new `operators as op` API, reworked recovery[^3]. |
| 0.17.0 | 2023-08-29 | Windowing and connector refinements. |
| 0.18.0 | 2023-12-20 | Continued dataflow/recovery API evolution. |
| 0.19.0 | 2024-03-20 | Operator and connector updates. |
| 0.20.0 | 2024-05-22 | — |
| 0.21.0 | 2024-08-06 | Latest minor line. |
| 0.21.1 | 2024-11-25 | Last tagged release to date[^3]. |
| — | 2025-05 | Company wound down; project becomes community-maintained[^2]. |

## References

[^1]: Timely Dataflow — Rust dataflow runtime underlying Bytewax's engine. https://github.com/TimelyDataflow/timely-dataflow
[^2]: Bytewax README, "Bytewax is now community-maintained" notice (as of May 2025) and issue #560. https://github.com/bytewax/bytewax/issues/560
[^3]: Bytewax releases/tags — latest 0.21.1 (2024-11-25); 0.16.0 (2023-04-19). https://github.com/bytewax/bytewax/releases
[^4]: Bytewax documentation — recovery and state snapshotting. https://docs.bytewax.io/
[^5]: PyPI package metadata (`requires_python >=3.8`). https://pypi.org/project/bytewax/

## Tags

python, rust, stream-processing, dataflow, stateful-streaming, timely-dataflow, kafka, data-engineering, real-time, event-processing, community-maintained
