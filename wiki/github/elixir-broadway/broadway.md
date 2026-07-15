# elixir-broadway/broadway

> Concurrent, multi-stage data ingestion pipelines for Elixir, built on GenStage with back-pressure baked in.

[GitHub repo](https://github.com/elixir-broadway/broadway) ·
[Official website](https://elixir-broadway.org) ·
[License: Apache-2.0](https://github.com/elixir-broadway/broadway/blob/main/LICENSE)

## Overview

Broadway is an Elixir library for consuming and processing data from external
sources — Amazon SQS, Apache Kafka, Google Cloud Pub/Sub, RabbitMQ, and
others — as long-lived, concurrent pipelines[^1]. It began at Plataformatec in
2019 and is now maintained by Dashbit, José Valim's company, the same group
behind Elixir itself and the GenStage/Flow lineage[^2].

Its defining idea is that you should not hand-assemble a GenStage topology of
producers and consumers to get a robust ingestion pipeline. Broadway takes a
declarative configuration — producers, processors, batchers, with concurrency
and demand settings — and builds the supervised stage graph for you, adding the
operational concerns most pipelines re-implement badly: back-pressure, automatic
acknowledgement, batching, graceful draining on shutdown, failure isolation,
rate limiting, partitioned ordering, and telemetry.

The central tension is that Broadway trades topology flexibility for
operational correctness. Its shape is fixed — producers feed processors, which
optionally feed batchers — and if your problem does not map onto that shape you
are meant to drop down to GenStage directly. In exchange, the hard parts
(demand-driven back-pressure so a slow consumer pauses the source instead of
exhausting memory, at-least-once delivery with correct ack timing) are handled
by the framework rather than by you.

## Getting Started

Add the dependency in `mix.exs`, alongside a producer library such as
`broadway_sqs`, `broadway_kafka`, or `broadway_rabbitmq`:

```elixir
def deps do
  [
    {:broadway, "~> 1.0"},
    {:broadway_sqs, "~> 0.7"}
  ]
end
```

A minimal pipeline consuming SQS and batching writes to S3:

```elixir
defmodule MyBroadway do
  use Broadway
  alias Broadway.Message

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwaySQS.Producer, queue_url: "https://sqs.us-east-2.amazonaws.com/1/my_queue"}
      ],
      processors: [default: [concurrency: 50]],
      batchers: [s3: [concurrency: 5, batch_size: 10, batch_timeout: 1_000]]
    )
  end

  # Called per message, concurrently across processors.
  def handle_message(_processor, %Message{} = message, _context) do
    message
    |> Message.update_data(&process/1)
    |> Message.put_batcher(:s3)
  end

  # Called once per full (or timed-out) batch.
  def handle_batch(:s3, messages, _batch_info, _context) do
    # write messages to S3 here
    messages
  end

  defp process(data), do: data
end
```

Add it to your supervision tree as `{MyBroadway, []}`. Messages that pass
through `handle_message`/`handle_batch` without being marked failed are
acknowledged back to the source automatically.

## Architecture / How It Works

Broadway is a thin, opinionated layer over **GenStage**, Elixir's demand-driven
producer/consumer abstraction[^2]. Every box in a Broadway pipeline is a
GenStage stage inside one supervision tree:

1. **Producers** pull from the source and emit `Broadway.Message` structs.
   Demand flows upstream — producers only fetch as many messages as downstream
   stages have asked for, which is where back-pressure comes from.
2. **Processors** run `handle_message/3` concurrently. `concurrency` sets how
   many processor stages exist; each pulls a bounded batch controlled by
   `min_demand`/`max_demand`.
3. **Batchers** (optional) group messages routed via `Message.put_batcher/2`
   and call `handle_batch/4` once a batch reaches `batch_size` or
   `batch_timeout` elapses — whichever comes first.
4. **Acknowledgers** run after a message exits the pipeline, telling the source
   it succeeded or failed. This is what makes delivery at-least-once.

Ordering is *not* guaranteed by default: with many concurrent processors,
messages are handled out of order. `partition_by` hashes a key so that all
messages for a given key land on the same processor, restoring per-key order at
the cost of parallelism. Failure is isolated per message — a raised exception
marks that message failed (routed to the optional `handle_failed/2`) without
taking down the stage; a genuine stage crash is restarted by the supervisor and
in-flight messages are acked as failed. `prepare_messages/2` lets you do a
single batched side-effect (e.g. one DB query) before per-message work.

## Production Notes

**Acknowledgement is at-least-once, so handlers must be idempotent.** A crash or
redelivery after partial work means the same message can be processed twice.
Broadway does not deduplicate for you; design writes to tolerate replays.

**Demand tuning drives both throughput and memory.** `max_demand` (default 10)
is the upper bound on messages a stage pulls at once. Large values increase
batching efficiency and memory footprint; small values reduce latency and RAM
but add coordination overhead. For batchers, `batch_size` must be reachable
within `batch_timeout` under real traffic or you will always flush on the
timeout and never fill a batch.

**Kafka has extra semantics via `broadway_kafka`.** Concurrency is bound to
partition count — one processor per assigned partition — and consumer-group
rebalancing reassigns partitions live. Offset commits are tied to
acknowledgement, so a poorly-behaved `handle_message` blocks offset progress and
can stall a partition. Ordering per partition holds; ordering across partitions
does not.

**Graceful shutdown drains in-flight work.** On termination Broadway stops
pulling new messages and lets outstanding ones finish within a shutdown timeout.
Set it long enough for your slowest batch, or you will ack-as-failed and
redeliver on the next boot.

**Rate limiting is producer-side.** The `rate_limiting` option caps messages per
interval — useful against downstream APIs with quotas — but it throttles intake,
not per-processor work, so it interacts with `max_demand` in non-obvious ways
under bursty load.

**Observability is via `:telemetry`.** Broadway emits telemetry events for
message and batch processing; there is no built-in dashboard. You wire them into
your metrics stack (`telemetry_metrics` + a reporter) yourself.

## When to Use / When Not

**Use when:**
- You need a continuous, long-running consumer of SQS/Kafka/Pub/Sub/RabbitMQ
  with back-pressure and at-least-once processing.
- You want batching, rate limiting, failure handling, and graceful draining
  without hand-writing a GenStage topology.
- You are already on the BEAM and want ingestion supervised like the rest of
  your app.

**Avoid when:**
- Your workload is a bounded dataset you want to map/reduce/join once — that is
  Flow's job, not Broadway's.
- You need durable, individually-scheduled, DB-backed jobs with retries — a job
  queue (Oban) fits better than a streaming pipeline.
- Your topology does not match producer → processor → batcher; use GenStage
  directly.
- You are not on the Erlang VM; the model does not transfer.

## Alternatives

- dashbitco/flow — in-VM data-parallel map/reduce with windows and joins; use for transforming a bounded or streaming *dataset* rather than running an operational ingestion pipeline.
- elixir-lang/gen_stage — the demand-driven producer/consumer primitive Broadway is built on; drop to it when your topology does not fit Broadway's fixed shape.
- oban-bg/oban — Postgres-backed job queue; use when you need durable, individually-retryable, scheduled background jobs instead of high-throughput stream consumption.
- apache/flink — JVM distributed stream processor with heavy stateful windowing and exactly-once; use at data-engineering scale beyond a single BEAM cluster.
- vectordotdev/vector — Rust observability/data pipeline agent; use to route logs, metrics, and events between systems declaratively without writing pipeline code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019 | First release under Plataformatec, built on GenStage[^1]. |
| 1.0.0 | 2020 | Stable API; stewardship moved to Dashbit[^3]. |
| 1.x | 2020–2026 | Ongoing: partitioning, rate limiting, telemetry, producer ecosystem growth. |

## References

[^1]: Broadway introduction and guides. https://hexdocs.pm/broadway/introduction.html
[^2]: GenStage — demand-driven producer/consumer stages for Elixir. https://hexdocs.pm/gen_stage
[^3]: Dashbit (formed 2020 from the Plataformatec team) — project steward. https://dashbit.co/
[^4]: Broadway official site. https://elixir-broadway.org

## Tags

elixir, erlang, beam, data-ingestion, data-processing, stream-processing, genstage, concurrency, back-pressure, message-queue, kafka, pipeline
