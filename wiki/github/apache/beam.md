# apache/beam

> A unified programming model for batch and streaming pipelines that run, unchanged, on Flink, Spark, Dataflow, and others.

[GitHub repo](https://github.com/apache/beam) ·
[Official website](https://beam.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/beam/blob/master/LICENSE)

## Overview

Apache Beam is a programming model plus a set of SDKs for writing data-parallel pipelines, and a set of runners that execute those pipelines on distributed backends. The premise: you write pipeline logic once against the Beam model, then choose an execution engine (a "runner") at submit time without rewriting. It came out of Google's internal data-processing lineage — MapReduce, FlumeJava, and MillWheel — formalized as the "Dataflow Model" in a 2015 VLDB paper, then donated to the Apache Software Foundation in 2016 and made a top-level project in 2017[^1].

The defining idea is that **batch and streaming are the same computation over a different completeness assumption**. Beam unifies them through four questions: *what* is computed (transforms), *where* in event time (windowing), *when* results are emitted (watermarks and triggers), and *how* refinements relate to prior emissions (accumulation mode)[^2]. This is genuinely more expressive than the "micro-batch vs. true streaming" framing of earlier systems, and it is also the source of Beam's steep conceptual load: event time vs. processing time, watermarks, allowed lateness, and trigger semantics must all be understood before non-trivial streaming works correctly.

The defining tension is **portability vs. the lowest common denominator**. The write-once promise only holds for features every target runner implements, and runners differ substantially in maturity and semantics. In practice Google Cloud Dataflow is the reference-quality runner (Beam and Dataflow share heritage and Google funds much of Beam's development); Flink and Spark runners are capable but have feature gaps and their own operational quirks. "Portable in theory" often becomes "tested on the one runner we actually ship to."

## Getting Started

```bash
# Python SDK
pip install apache-beam

# Java: add org.apache.beam:beam-sdks-java-core + a runner artifact via Maven/Gradle
# Go:   go get github.com/apache/beam/sdks/v2/go/pkg/beam
```

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

# DirectRunner by default; swap --runner=FlinkRunner / DataflowRunner to change engine
with beam.Pipeline(options=PipelineOptions()) as p:
    (
        p
        | "Read" >> beam.io.ReadFromText("input.txt")
        | "Split" >> beam.FlatMap(lambda line: line.split())
        | "Pair" >> beam.Map(lambda w: (w, 1))
        | "Count" >> beam.CombinePerKey(sum)
        | "Write" >> beam.io.WriteToText("output")
    )
```

## Architecture / How It Works

A Beam program builds a directed acyclic graph of `PTransform`s connected by `PCollection`s (immutable, possibly-unbounded collections of timestamped elements). Nothing executes when you call the transforms — you are constructing a graph. A `PipelineRunner` translates that graph into engine-specific operations at submit time.

The core primitives:

- **`PCollection`** — a distributed dataset, bounded (batch source) or unbounded (streaming source). Every element carries an event-time timestamp and a window assignment.
- **`PTransform`** — a node in the graph. The universal primitive is `ParDo` (a parallel per-element `DoFn`, analogous to a flatMap with lifecycle hooks and side inputs/outputs). `GroupByKey` is the other foundational transform; most higher-level transforms (`Combine`, joins, `CoGroupByKey`) desugar to `ParDo` + `GroupByKey`.
- **Windowing + triggers** — windows partition data along event time (fixed, sliding, sessions, global). Watermarks estimate input completeness; triggers decide when a window's pane fires (on watermark, on element count, on processing-time delay, or combinations). Accumulation mode controls whether re-fired panes replace or add to prior output.
- **State and timers** — `DoFn`s can hold per-key-and-window state and set event/processing-time timers, enabling arbitrary stateful streaming logic below the windowing layer.

**Portability framework.** Older runners embedded each SDK's execution directly (the "classic" path, JVM-only). The newer **portability layer** decouples SDK from runner via language-neutral protobuf pipeline definitions and two gRPC contracts: the **Runner API** (the serialized pipeline graph) and the **Fn API** (how a runner drives an SDK "harness" to execute user code). Under this model, user code runs in a per-language **SDK harness** — usually a Docker container — that the runner communicates with over gRPC. This is what makes Python and Go pipelines runnable on JVM engines like Flink, and what makes cross-language transforms (e.g. calling a Java IO from Python) possible. It also introduces containers, gRPC round-trips, and element serialization into the hot path.

`PrismRunner` is a portable local runner (Go-based) intended as the default local/testing runner, replacing much of `DirectRunner`'s role for portability-accurate local execution.

## Production Notes

**Runner parity is not uniform.** The write-once model leaks. Trigger semantics, late-data handling, exactly-once guarantees, and IO connector support vary by runner. Code validated on `DirectRunner` can behave differently on Flink or Spark, and features work on Dataflow that lag elsewhere. Test on the runner you deploy to — the local runner is not a faithful proxy for distributed timing or watermark behavior.

**Python performance and packaging.** The Python SDK runs user code in the SDK harness with per-element serialization (Coders) crossing the Fn API boundary; throughput is well below native Java on the same runner. Managing worker dependencies is a recurring pain: you must ship your Python environment to workers via `--requirements_file`, `--setup_file`, `--extra_packages`, or a custom SDK container image, and version skew between the launch environment and the worker image causes pickling/import failures that surface only at runtime. RunInference and other ML transforms amplify the container-image discipline required.

**The DirectRunner is a correctness tool, not a performance one.** It deliberately does extra work (encoding round-trips, element reordering, immutability checks) to catch bugs; it is not representative of production throughput and should never be benchmarked as such.

**`GroupByKey` and hot keys.** As in any shuffle-based system, skewed keys create stragglers. Beam gives you `Combine` (associative/commutative pre-aggregation) to push reduction before the shuffle, but hot-key mitigation, fanout, and windowing granularity are on you.

**Streaming semantics are subtle.** Watermarks are estimates, not guarantees; allowed lateness, trigger accumulation mode, and windowing interact in ways that silently drop or double-count data if misconfigured. Draining/updating a running streaming pipeline while preserving in-flight state is runner-specific and a common operational sharp edge (Dataflow supports pipeline update; other runners rely on their own savepoint/checkpoint mechanics).

**Version coupling.** A Beam release targets specific runner-engine versions (e.g. a given Beam version supports a bounded range of Flink or Spark versions). Upgrading Beam and upgrading your cluster engine are entangled; check the compatibility matrix before bumping either.

## When to Use / When Not

**Use when:**
- You need one pipeline definition that can move between engines or hedge against engine lock-in.
- You are on Google Cloud Dataflow — Beam is the native SDK and the experience is best-supported there.
- You need genuinely unified batch + streaming semantics (event-time windowing, triggers, exactly-once) rather than bolting streaming onto a batch tool.
- You want to use Python or Go transforms on a JVM execution engine via the portability layer.

**Avoid when:**
- You are committed to a single engine and want its full feature set and lowest latency — native Flink DataStream or Spark Structured Streaming will expose capabilities Beam's abstraction hides.
- Your workload is straightforward SQL/batch analytics — a warehouse or Spark SQL is simpler than Beam's model.
- Your data lives entirely in Kafka and you want a library, not a cluster — Kafka Streams is lighter.
- You want minimal conceptual overhead — Beam's windowing/trigger/watermark model has a real learning curve before non-trivial streaming is correct.

## Alternatives

- apache/flink — use directly when you commit to one streaming engine and want the lowest latency and full native feature surface.
- apache/spark — use for batch-heavy analytics, SQL, and MLlib when portability across engines is not a requirement.
- apache/kafka — Kafka Streams: use when all your data is already in Kafka and you want an embedded library instead of a separate processing cluster.
- spotify/scio — a Scala DSL on top of Beam; use when you want Beam's model with far better Scala ergonomics than the raw Java SDK.
- bytewax/bytewax — use for pure-Python stream processing (Rust core, Timely Dataflow) when you want to avoid the JVM and container harness entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-02 | Donated to Apache as "Apache Beam" (incubating), from Google's Cloud Dataflow SDK[^1]. |
| — | 2017-01 | Graduated to a top-level Apache project. |
| 2.0.0 | 2017-05 | First stable, API-guaranteed Apache release; the project has stayed on the 2.x line since[^3]. |
| — | 2018–2019 | Portability framework (Runner API / Fn API) matures; Python and Go on Flink/Spark become viable. |
| — | 2022 | Go SDK reaches general availability; RunInference brings first-class ML inference transforms. |
| — | 2024–2025 | PrismRunner promoted as the portable default local runner; managed/turnkey IO and ML transforms expand. |

## References

[^1]: Apache Software Foundation, "The Apache Software Foundation Announces Apache Beam as a Top-Level Project" — 2017. https://blogs.apache.org/foundation/entry/the-apache-software-foundation-announces75
[^2]: Tyler Akidau et al., "The World Beyond Batch: Streaming 101 / 102" — what/where/when/how framing. https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/
[^3]: Apache Beam release documentation and downloads. https://beam.apache.org/get-started/downloads/
[^4]: Akidau et al., "The Dataflow Model" — VLDB 2015. http://www.vldb.org/pvldb/vol8/p1792-Akidau.pdf

## Tags

data-processing, batch, streaming, big-data, java, python, golang, pipeline, distributed-systems, apache, dataflow, etl
