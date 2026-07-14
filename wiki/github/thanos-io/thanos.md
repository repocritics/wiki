# thanos-io/thanos

> A set of components that bolt long-term object-storage retention and a global query view onto existing Prometheus deployments.

[GitHub repo](https://github.com/thanos-io/thanos) ·
[Official website](https://thanos.io) ·
[License: Apache-2.0](https://github.com/thanos-io/thanos/blob/main/LICENSE)

## Overview

Thanos is a CNCF Incubating project that extends Prometheus with three things vanilla Prometheus lacks: unlimited retention (via object storage), a global query view across many Prometheus servers, and deduplication of metrics scraped by highly-available Prometheus pairs[^1]. It was announced in 2017 by engineers at Improbable and reached its first tagged release in 2018[^2]. Crucially, it reuses the Prometheus 2.x TSDB block format verbatim — Thanos does not reimplement storage, it ships blocks to a bucket and reads them back.

The design is deliberately UNIX-flavoured: a handful of small single-purpose binaries (the same `thanos` binary run with different subcommands) that talk to each other over a gRPC "StoreAPI" and share nothing but the object store[^3]. This is the project's defining tension. The component model is composable and lets you scale each concern independently, but it pushes real operational complexity onto the operator: you assemble Sidecar, Store Gateway, Querier, Compactor, and (optionally) Receiver yourself, wire their gRPC endpoints, and reason about which component owns which failure. Thanos gives you primitives, not a turnkey system.

It competes in a crowded "Prometheus long-term storage" space against Cortex, Grafana Mimir (a Cortex fork), and VictoriaMetrics. Thanos's differentiator is that it stays closest to Prometheus semantics and treats object storage as the only hard dependency, at the cost of higher query latency than the index-in-a-database designs.

## Getting Started

Thanos is a single Go binary; most operators run the container image from `quay.io/thanos/thanos`. A minimal setup runs a sidecar next to Prometheus and a querier in front of it:

```bash
# 1. Sidecar next to a Prometheus that has --storage.tsdb.max-block-duration=2h
thanos sidecar \
  --tsdb.path=/prometheus \
  --prometheus.url=http://localhost:9090 \
  --grpc-address=0.0.0.0:10901 \
  --objstore.config-file=bucket.yml

# 2. Querier fans out over one or more StoreAPI endpoints
thanos query \
  --http-address=0.0.0.0:9091 \
  --endpoint=localhost:10901
```

```yaml
# bucket.yml — object storage is the only external dependency
type: S3
config:
  bucket: thanos-metrics
  endpoint: s3.us-east-1.amazonaws.com
  region: us-east-1
```

The querier exposes the Prometheus HTTP query API on `:9091`, so Grafana points at it exactly as it would at Prometheus.

## Architecture / How It Works

Thanos is a set of stateless-ish processes coordinated through a bucket and gRPC:

- **Sidecar** — runs beside each Prometheus, exposes its recent (in-memory / local-TSDB) data over StoreAPI, and uploads completed 2h TSDB blocks to object storage. This is the "pull" ingestion model and requires no change to Prometheus's own operation.
- **Store Gateway** — exposes the *historical* blocks sitting in object storage over the same StoreAPI. It downloads and caches block index headers so it can answer queries without pulling whole blocks.
- **Querier (`thanos query`)** — the fan-out node. It implements the Prometheus query API, discovers all StoreAPI endpoints (sidecars, store gateways, receivers), fans a query out to all of them, and merges + deduplicates the results. Deduplication of HA Prometheus pairs happens here, keyed by a replica label.
- **Compactor** — an offline batch process that compacts small blocks into larger ones, applies retention/deletion, and produces **downsampled** resolutions (raw → 5m → 1h) so that long-range queries scan far fewer samples.
- **Receiver (`thanos receive`)** — the alternative "push" ingestion path: it implements Prometheus remote-write, so tenants push samples to Thanos instead of Thanos scraping them. This is how multi-tenant and cross-cluster setups are usually built.
- **Query Frontend** — an optional caching/splitting proxy in front of the querier that splits long queries by time and caches results.

The StoreAPI is the unifying abstraction: every data source — recent, historical, pushed — looks identical to the querier[^3]. Because everything reuses Prometheus's block format, there is no separate ingestion database to corrupt or migrate; the bucket *is* the database.

## Production Notes

The differentiator between "it works in a demo" and "it works at scale" is a short list of sharp edges every Thanos operator eventually hits:

- **The Compactor must be a singleton per bucket.** Running two compactors against the same object-storage bucket will corrupt blocks and produce overlapping-block errors that are painful to unwind. This is the single most-cited Thanos footgun[^4]. It also means the compactor is a scaling bottleneck; the mitigation is vertical scaling plus label-based sharding across multiple compactors, each owning a disjoint slice of blocks.
- **Store Gateway memory and index cache.** The store gateway holds block index headers in memory; with large numbers of blocks or high cardinality it becomes memory-hungry, and cold queries stall while it lazy-loads index headers. Production setups front it with an index cache and a caching bucket (in-memory or memcached/Redis) and shard the gateway by time range or hashmod.
- **Query latency is fan-out latency.** A global query is only as fast as the slowest StoreAPI endpoint it touches. `--query.partial-response` lets the querier return partial results when a store is down, which is often the right tradeoff but changes correctness semantics — a graph can silently miss a series because one store timed out.
- **Object-storage cost is request cost, not just bytes.** Store gateway and compactor generate a large volume of GET/LIST operations. On S3 the API request bill and, for cross-AZ setups, egress can dominate storage cost; the caching bucket exists largely to suppress this.
- **Downsampling is not free and not lossless for all functions.** Downsampled series speed up long-range queries but the compactor keeps raw + 5m + 1h, multiplying storage. Queries that need raw resolution (e.g. `rate()` over short windows) must still hit raw blocks.
- **Deduplication has counter edge cases.** Penalty-based dedup across HA replicas generally works, but at the exact moment of failover between replicas you can see small artifacts in counter rates. Consistent, stable `replica` external labels are essential.
- **Receive is stateful and harder to operate** than the sidecar path: it uses a hashring for tenant/series distribution, and resharding the hashring is a genuine operational event. Prefer sidecars unless you specifically need remote-write ingestion.

Thanos has never shipped a 1.0 — it remains on a `0.x` line with a minor release roughly every six weeks[^2]. In practice the project treats `main` as stable and the format is mature, but the pre-1.0 label is a real thing to note when arguing for adoption in conservative shops.

## When to Use / When Not

**Use when:**
- You already run Prometheus and want cheap long-term retention without changing how Prometheus scrapes.
- You need a single global query view across many clusters/regions.
- You want object storage (S3/GCS/Azure) to be the only hard external dependency.
- You run HA Prometheus pairs and need transparent deduplication.

**Avoid when:**
- You want a single turnkey binary — VictoriaMetrics is dramatically simpler to operate.
- You need consistently low-latency queries over huge historical ranges; the fan-out + object-store read path is inherently slower than index-in-a-database designs.
- You have a small single-Prometheus setup where local retention already suffices — Thanos is overhead you don't need.
- Your team can't dedicate someone to operating a distributed, multi-component system.

## Alternatives

- VictoriaMetrics/VictoriaMetrics — use instead when you want a much simpler, faster-to-operate single-system TSDB and can accept some deviation from Prometheus semantics.
- grafana/mimir — use instead when you want a horizontally-scalable, multi-tenant, managed-friendly system with a microservices model and are fine with a heavier deployment.
- cortexproject/cortex — the original horizontally-scalable Prometheus backend (Mimir is its fork); use when you specifically need Cortex's remote-write-first, database-backed index model.
- prometheus/prometheus — use alone when your retention and single-cluster scale fit in local TSDB and you don't need a global view.
- m3db/m3 — use when you want a purpose-built distributed TSDB and are prepared for its operational weight.

## History

| Version | Date | Notes |
|---------|------|-------|
| Announced | 2017 | Introduced by Improbable engineers; project open-sourced[^2]. |
| 0.1.0 | 2018 | First tagged release: sidecar, store, query, compact[^2]. |
| CNCF Sandbox | 2019-08 | Accepted into the CNCF at Sandbox maturity[^1]. |
| Receive | 2019 | `thanos receive` adds a remote-write push ingestion path. |
| CNCF Incubating | 2020 | Promoted to Incubating maturity[^1]. |
| Query Frontend | 2020 | Caching/splitting proxy for the querier introduced. |
| 0.x (ongoing) | 2026 | Still pre-1.0; ~6-week minor cadence, `main` treated as stable[^2]. |

## References

[^1]: CNCF project listing — Thanos (Incubating). https://www.cncf.io/projects/thanos/
[^2]: Thanos releases and release process. https://github.com/thanos-io/thanos/releases and https://github.com/thanos-io/thanos/blob/main/docs/release-process.md
[^3]: Thanos design documentation (components, StoreAPI). https://thanos.io/tip/thanos/design.md/
[^4]: Thanos Compactor documentation — single-instance requirement. https://thanos.io/tip/components/compact.md/

## Tags

go, prometheus, observability, monitoring, metrics, time-series, object-storage, cncf, high-availability, long-term-storage, distributed-systems
