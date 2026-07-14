# grafana/mimir

> Horizontally scalable, multi-tenant long-term storage for Prometheus, built on object storage.

[GitHub repo](https://github.com/grafana/mimir) ·
[Official website](https://grafana.com/oss/mimir/) ·
[License: AGPL-3.0](https://github.com/grafana/mimir/blob/main/LICENSE)

## Overview

Grafana Mimir is a time-series database that takes the data Prometheus scrapes and stores it durably, cheaply, and at a scale a single Prometheus cannot reach. Grafana Labs announced it in March 2022 as a hard fork of Cortex, carrying over that project's distributed architecture and its Apache-2.0 heritage into an AGPL-3.0 codebase[^1]. It is aimed at operators who already run Prometheus, have outgrown one server, and need a global query view, multi-year retention, and per-team isolation without gluing those together by hand.

The defining tension is between the promise ("just one binary in monolithic mode") and the reality of running it at scale. Mimir compiles to a single Go binary that can run as a self-contained instance for evaluation, but its production identity is a fleet of stateless-and-stateful microservices — distributor, ingester, querier, query-frontend, store-gateway, compactor, ruler, alertmanager — coordinated through a hash ring[^2]. The README's headline internal-testing figure is 1 billion active series[^3]; getting anywhere near that means operating the disaggregated topology, an object store, and the ring, not the one-binary demo.

Mimir does not scrape metrics itself. It sits behind Prometheus (via remote-write) or an OpenTelemetry Collector (via OTLP), and Grafana or any PromQL client queries it back. It is infrastructure for an existing observability stack, not a turnkey monitoring product.

## Getting Started

Single-binary evaluation with Docker:

```bash
docker run --rm --name mimir -p 9009:9009 \
  grafana/mimir:latest \
  -config.file=/etc/mimir/demo.yaml
```

Point Prometheus at it with remote-write in `prometheus.yml`:

```yaml
remote_write:
  - url: http://localhost:9009/api/v1/push
    headers:
      X-Scope-OrgID: demo   # tenant id; required unless auth is disabled
```

Query it back over the Prometheus-compatible API:

```bash
curl -H 'X-Scope-OrgID: demo' \
  'http://localhost:9009/prometheus/api/v1/query?query=up'
```

The `X-Scope-OrgID` header is the tenant selector and is mandatory whenever multi-tenancy is enabled (the default). A missing header is the single most common first-run error.

## Architecture / How It Works

Mimir splits the write path and read path across independently scalable components, all discovering each other through a **hash ring** backed by a gossip protocol (memberlist by default, or Consul/etcd)[^2]:

- **Distributor** — receives remote-write/OTLP, validates and rate-limits per tenant, then replicates each series to multiple ingesters (default replication factor 3) chosen by consistent hashing.
- **Ingester** — holds recent samples in memory and a write-ahead log, and periodically flushes them as immutable **TSDB blocks** (the same on-disk format Prometheus uses) to object storage. Ingesters are the stateful, memory-hungry heart of the write path.
- **Compactor** — merges and deduplicates blocks in object storage over time, reducing block count and query fan-out.
- **Store-gateway** — serves queries against historical blocks in object storage, keeping block index-headers to avoid full downloads.
- **Querier / query-frontend** — the query-frontend splits, aligns, caches, and queues queries; queriers fan out to ingesters (recent data) and store-gateways (historical data) and merge results.
- **Ruler** and **Alertmanager** — evaluate recording/alerting rules and handle alert routing, mirroring Prometheus and the standalone Alertmanager.

Object storage (S3, GCS, Azure Blob, Swift, or any S3-compatible target) is the durability layer; there is no separate database. All long-term data lives as TSDB blocks in a bucket, which is why storage cost tracks object-store pricing rather than provisioned disks.

Two features distinguish Mimir from its Cortex ancestor: a rewritten **blocks-only storage** path (Cortex's older chunks storage was dropped) and **query sharding**, where the query-frontend splits a single high-cardinality PromQL query into shards executed in parallel across queriers[^2]. The shared low-level libraries (ring, memberlist wiring, service lifecycle) live in Grafana's `dskit`, also used by Loki and Tempo.

## Production Notes

**Deployment modes are a real fork in the road.** *Monolithic* runs every component in one process (`-target=all`) — fine for small clusters, but you scale the whole thing together. *Microservices* runs each `-target` as its own deployment and is what the 1-billion-series claim assumes. There is also a *read-write* mode that collapses components into three groups. Most teams run Mimir on Kubernetes via the official Helm chart or the Jsonnet libraries; running it well outside Kubernetes is possible but under-trodden.

**Ingesters are the operational pain point.** They hold hours of samples in RAM before flushing, so memory sizing, the replication factor, and graceful shutdown all matter. A rollout that terminates ingesters faster than they can hand off risks data loss for unflushed samples; Mimir provides shutdown/handover procedures precisely because naive `kubectl delete pod` is unsafe. Memory is typically the binding constraint on how many active series a cluster holds.

**Object storage is on the query hot path.** Historical queries hit the store-gateway, which hits the bucket. Under-provisioned store-gateway memory (for index-headers) or high object-store latency shows up directly as slow queries. The store-gateway also shards blocks across replicas via the ring, so scaling it is not just "add memory."

**Multi-tenancy needs a trusted proxy.** Mimir itself does no authentication — it trusts the `X-Scope-OrgID` header. In production you put an authenticating gateway (nginx, Grafana Enterprise Metrics, or a custom proxy) in front to set that header per tenant. Exposing Mimir directly is an isolation hole.

**Cardinality is the cost driver.** Per-tenant limits (max series, max samples, ingestion rate, query length) exist because one tenant's label explosion can exhaust ingester memory for everyone. Operators spend real time tuning `limits` and watching per-tenant cardinality; the defaults are conservative and will reject load until raised.

**Upgrades.** Grafana documents upgrade paths and Mimir supports rolling, zero-downtime upgrades across its components, but the ring and ingester statefulness mean you follow the ordered procedure rather than restarting everything at once. Migrations from Cortex and from Thanos/Prometheus have dedicated guides[^4].

## When to Use / When Not

**Use when:**
- You run multiple Prometheus servers and need one global, long-retention query view.
- You want metric storage costs tied to cheap object storage rather than provisioned disk.
- You need hard multi-tenant isolation (per-team limits, separate data) on shared infrastructure.
- You are on Kubernetes and can adopt the Helm/Jsonnet operational model.

**Avoid when:**
- One Prometheus (or Prometheus + a little retention) already covers you — Mimir's operational surface is large.
- You want a single-process TSDB with minimal moving parts — VictoriaMetrics is far simpler to run.
- You cannot provide object storage, or you are not on Kubernetes and don't want to build the tooling yourself.
- AGPL-3.0 is incompatible with your distribution model (note this governs the server, not your metrics or PromQL usage).

## Alternatives

- VictoriaMetrics/VictoriaMetrics — use instead when you want dramatically lower operational complexity and memory footprint for the same "long-term Prometheus storage" job, at the cost of a smaller ecosystem and a different (single-vendor) architecture.
- thanos-io/thanos — use instead when you prefer a sidecar-based model that augments existing Prometheus servers rather than replacing their storage path, and want a CNCF-governed project.
- cortexproject/cortex — the project Mimir forked from; use only if you have an existing Cortex deployment or need its specific governance, since Mimir is where Grafana's active investment went.
- prometheus/prometheus — use instead when a single server's retention and scale are enough; Mimir is the answer only once you outgrow it.
- grafana/loki — not an alternative for metrics; the sibling project for logs, sharing Mimir's `dskit` foundation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 | 2022-03-30 | Initial public release; hard fork of Cortex, blocks-only storage, query sharding[^1]. |
| 2.x | 2022–2025 | Regular minor releases: OTLP ingestion, native histograms, ruler and compactor improvements. |
| 2.16+ | 2026 | Active development on `main`; ongoing OTLP/native-histogram and query-engine work[^5]. |

(Mimir continued Cortex's version numbering, starting at 2.0 rather than 1.0. Only the initial-release date is asserted with high confidence; intermediate feature dates are omitted rather than guessed.)

## References

[^1]: Grafana Labs, "Announcing Grafana Mimir" — 2022-03-30. https://grafana.com/blog/2022/03/30/announcing-grafana-mimir/
[^2]: Grafana Mimir architecture documentation. https://grafana.com/docs/mimir/latest/references/architecture/
[^3]: Grafana Mimir README (1 billion active series internal-testing figure). https://github.com/grafana/mimir
[^4]: Grafana Mimir migration guides (from Cortex, and from Thanos/Prometheus). https://grafana.com/docs/mimir/latest/set-up/migrate/
[^5]: Grafana Mimir releases. https://github.com/grafana/mimir/releases

## Tags

go, prometheus, tsdb, observability, metrics, time-series, long-term-storage, multi-tenant, object-storage, opentelemetry, kubernetes, distributed-systems
