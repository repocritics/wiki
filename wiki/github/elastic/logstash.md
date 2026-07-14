# elastic/logstash

> A JVM-based server-side data pipeline — inputs, filters, outputs joined by a config DSL — that ingests, transforms, and ships logs and events, most often into Elasticsearch.

[GitHub repo](https://github.com/elastic/logstash) ·
[Official website](https://www.elastic.co/products/logstash) ·
License: Apache-2.0 (core) + Elastic License 2.0 (x-pack)[^1]

## Overview

Logstash is the "L" in the classic ELK stack (Elasticsearch, Logstash, Kibana) and one of the oldest tools in the log-processing space, first released by Jordan Sissel in 2013 and later folded into Elastic[^2]. It is a data-processing pipeline: it collects events from many sources, mutates them through a chain of filters, and emits them to one or more destinations. Elasticsearch is the default and best-supported sink, but Logstash predates the tight Elastic coupling and can write to Kafka, S3, relational databases, other Logstash instances, or plain files.

Its defining characteristic is the **filter stage**, and specifically `grok` — regex-based extraction of structured fields from unstructured log lines. This is what made Logstash the standard way to turn messy syslog and application logs into indexed JSON in the early 2010s. It is also its defining liability: Logstash runs on the JVM (via JRuby plus a Java core), so it carries a large memory footprint and slow startup relative to the lightweight shippers Elastic later built to sit in front of it.

The project today lives in an awkward middle. Elastic's own guidance increasingly routes simple use cases through **Beats** (or Elastic Agent) shipping directly to **Elasticsearch ingest pipelines**, bypassing Logstash entirely. Logstash remains the tool of choice when you need heavier transformation, buffering, fan-out to non-Elastic sinks, or protocol translation that ingest nodes cannot do. Understanding whether your workload actually needs Logstash — versus a shipper plus an ingest pipeline — is the single most important design decision in this ecosystem.

## Getting Started

Logstash is normally run as a released binary/package, not built from source. A pipeline is defined in a `.conf` file:

```ruby
# pipeline.conf — read stdin, parse an Apache log line, print structured JSON
input {
  stdin { }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

output {
  stdout { codec => rubydebug }
  # elasticsearch { hosts => ["http://localhost:9200"] index => "logs-%{+YYYY.MM.dd}" }
}
```

```sh
bin/logstash -f pipeline.conf
# or a throwaway one-liner:
bin/logstash -e 'input { stdin {} } output { stdout {} }'
```

Plugins ship as Ruby gems and are managed with `bin/logstash-plugin install logstash-filter-dissect`.

## Architecture / How It Works

A Logstash pipeline is three ordered stages — **input → filter → output** — connected by an internal queue.

- **Inputs** poll or listen for events (`beats`, `file`, `kafka`, `http`, `jdbc`, `syslog`). Each runs in its own thread.
- **Filters** transform events sequentially. `grok` (regex), `dissect` (delimiter-based, faster), `mutate`, `date`, `geoip`, `json`, and `ruby` (arbitrary code) are the workhorses.
- **Outputs** write to sinks. Multiple outputs fan out the same event.

Between input and filter/output sits a **queue**. The default is an in-memory bounded queue; the alternative is the **persistent queue (PQ)**, which writes to disk so events survive a crash and provides at-least-once delivery within Logstash[^3]. Events that repeatedly fail processing can be routed to a **dead letter queue (DLQ)** for later inspection rather than being dropped.

The execution model has been rewritten over time. Early Logstash was JRuby end to end; the pipeline execution engine was reimplemented in Java (the "Java execution engine," default since 7.0) for throughput, while plugins remain Ruby gems calling into the Java core[^4]. This hybrid — JRuby plugins on a Java engine on the JVM — is why the codebase is split across `logstash-core` (Java) and Ruby, and why the plugin ecosystem lives in a separate `logstash-plugins` GitHub org with 200+ independently versioned gems.

Filter ordering is deterministic within a pipeline, but with `pipeline.workers > 1` events are processed in parallel across worker threads, so **global event ordering is not guaranteed**. Logstash also supports **multiple pipelines** in one process (`pipelines.yml`) and **pipeline-to-pipeline** communication for building fan-in/fan-out topologies inside a single instance.

## Production Notes

**Memory and JVM tuning.** Logstash is heap-hungry. The dominant knobs are `-Xms`/`-Xmx` (set them equal, avoid swapping), `pipeline.workers` (defaults to CPU core count), and `pipeline.batch.size`. Under-provisioned heap manifests as GC pauses that stall the whole pipeline. This weight is the reason Beats exists.

**Grok is the classic footgun.** Poorly anchored grok patterns backtrack catastrophically on non-matching lines and can peg a CPU core. Prefer `dissect` for fixed-delimiter formats, anchor patterns, and use `grok`'s `timeout_millis` on untrusted input. A single bad pattern in a hot path is a common cause of "Logstash fell behind."

**Persistent queue trade-offs.** The PQ gives durability and absorbs downstream backpressure (e.g., Elasticsearch being slow), but adds disk I/O and its own tuning (`queue.max_bytes`, `queue.checkpoint.writes`). It guarantees at-least-once, not exactly-once — duplicates are possible after a crash, so downstream consumers should be idempotent (use document IDs in Elasticsearch).

**Backpressure is internal, not end-to-end.** When outputs slow down, the queue fills and inputs block — but push-based inputs (syslog/UDP) can drop data upstream regardless. Pull-based inputs (Kafka, Beats with acks) are safer for lossless delivery.

**Monitoring.** Logstash exposes a node stats API on port 9600 (`_node/stats`, `_node/hot_threads`) for pipeline throughput, queue depth, and JVM health. This is essential — Logstash gives few external signals when it silently falls behind.

**Upgrade friction.** Versioning is aligned with the Elastic Stack, so Logstash 8.x pairs with Elasticsearch 8.x. Plugin gems version independently and occasionally break across major upgrades; pin plugin versions and test pipelines before upgrading. The 6.x→7.x transition changed the default execution engine; 7.x→8.x tightened security defaults (TLS, security on by default in the stack).

## When to Use / When Not

**Use when:**
- You need real transformation, enrichment, or parsing beyond what an Elasticsearch ingest pipeline can do.
- You need buffering/durability (persistent queue) to absorb downstream outages.
- You are fanning one event stream out to multiple heterogeneous sinks, or translating between protocols (e.g., Kafka in, Elasticsearch + S3 out).
- You already run the Elastic Stack and want the best-integrated pipeline for it.

**Avoid when:**
- You only need to ship logs unchanged — use a Beat / Elastic Agent / Fluent Bit instead; Logstash's JVM overhead is wasted.
- Your transforms are simple field renames or `grok`s that Elasticsearch ingest pipelines can handle in the cluster.
- You are resource-constrained at the edge (containers, IoT) — the JVM footprint is a poor fit.
- You want a vendor-neutral pipeline and are wary of Elastic's licensing direction (x-pack features fall under the Elastic License, not Apache-2.0).

## Alternatives

- fluent/fluentd — CNCF-graduated Ruby log collector; use instead when you want a vendor-neutral, cloud-native standard with a large plugin ecosystem.
- fluent/fluent-bit — C-based lightweight collector; use instead at the edge / in Kubernetes where memory footprint matters.
- vectordotdev/vector — Rust pipeline (Datadog); use instead when you want Logstash-class transformation at a fraction of the memory and no JVM.
- elastic/beats — lightweight single-purpose shippers (Filebeat, Metricbeat); use instead when you only forward data and do transforms in Elasticsearch ingest pipelines.
- apache/nifi — visual, flow-based data routing; use instead for complex, GUI-managed enterprise dataflows across many systems.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013-01 | First stable release under Jordan Sissel[^2]. |
| 2.0 | 2015-10 | Pipeline reliability work, acked in-memory queue. |
| 5.0 | 2016-10 | Unified Elastic Stack versioning (3.x/4.x skipped); persistent queue beta. |
| 6.0 | 2017-11 | Persistent queue GA, multiple pipelines, `pipelines.yml`. |
| 7.0 | 2019-04 | Java execution engine as default. |
| 8.0 | 2022-02 | Stack security-on-by-default alignment, ECS defaults. |
| 9.0 | 2025-04 | Aligned with Elastic Stack 9.x major. |

## References

[^1]: Logstash LICENSE.txt — core is Apache License 2.0; the `x-pack` folder is under the Elastic License 2.0; `-oss` artifacts are Apache-2.0 only. https://github.com/elastic/logstash/blob/main/LICENSE.txt
[^2]: Logstash originated as an independent project by Jordan Sissel before Elastic; project history and downloads. https://www.elastic.co/products/logstash
[^3]: Elastic docs, "Persistent queues (PQ)" — durability and at-least-once delivery. https://www.elastic.co/guide/en/logstash/current/persistent-queues.html
[^4]: Elastic docs, "How Logstash Works" — inputs, filters, outputs, execution model. https://www.elastic.co/guide/en/logstash/current/pipeline.html

## Tags

java, jruby, log-processing, etl, data-pipeline, elastic-stack, observability, streaming, grok, elasticsearch, logging, jvm
