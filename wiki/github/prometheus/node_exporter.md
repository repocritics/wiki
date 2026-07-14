# prometheus/node_exporter

> The reference Prometheus agent for *NIX host and hardware metrics, exposing kernel counters over HTTP for scraping.

[GitHub repo](https://github.com/prometheus/node_exporter) ·
[Official website](https://prometheus.io/) ·
[License: Apache-2.0](https://github.com/prometheus/node_exporter/blob/master/LICENSE)

## Overview

`node_exporter` is the canonical way to get machine-level metrics — CPU, memory,
disk, filesystem, network, load, and dozens of narrower kernel and hardware
sources — into Prometheus. It runs as a small Go daemon on each host, reads
`/proc`, `/sys`, and platform sysctls through a set of pluggable *collectors*,
and serves the current values as a Prometheus text exposition on `:9100/metrics`.
It holds no history of its own; Prometheus pulls the endpoint on an interval and
does the storage. It is one of the oldest exporters in the ecosystem (first
commit 2013) and is effectively the default host agent wherever Prometheus is
used[^1].

Its scope is deliberately narrow and that is the recurring tension. It monitors
the *host*, not applications, not containers, and not Windows — the project
explicitly redirects Windows users to `prometheus-community/windows_exporter` and
GPU users to `NVIDIA/dcgm-exporter`[^2]. It exports what the kernel already
counts and refuses to editorialize: no thresholds, no alerting, no dashboards.
That minimalism is why it is stable and ubiquitous, and also why a real
deployment always pulls in several other exporters and a Grafana/alerting layer
around it.

The other defining fact is that it is a pull-model, host-local agent. Each
machine runs its own copy, listens on a port, and is discovered and scraped by
Prometheus. There is no push, no central agent config, and — until relatively
recently — no built-in authentication or TLS.

## Getting Started

```bash
# Prebuilt binary (Linux amd64) — no dependencies, static-ish Go binary
curl -fsSL -O https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-*.linux-amd64.tar.gz
tar xzf node_exporter-*.linux-amd64.tar.gz
./node_exporter-*/node_exporter        # serves on 0.0.0.0:9100/metrics
```

Point Prometheus at it:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: node
    static_configs:
      - targets: ["host-a:9100", "host-b:9100"]
```

In production it is normally run under systemd as an unprivileged `node_exporter`
user. Distro packages exist (EPEL ships `golang-github-prometheus-node-exporter`;
the Prometheus Community Ansible role automates installs)[^2].

## Architecture / How It Works

The daemon is a thin HTTP server wrapping a registry of collectors. On each
scrape it runs every *enabled* collector concurrently, each of which reads its
source (`/proc/stat`, `/proc/meminfo`, `/sys/class/...`, a sysctl, etc.) and
emits gauges/counters. Nothing is cached between scrapes — every request re-reads
the kernel. Collectors are toggled with `--collector.<name>` /
`--no-collector.<name>`, and a subset support include/exclude regex flags (e.g.
`--collector.filesystem.mount-points-exclude`) to control cardinality[^2].

Collectors fall into three groups: **enabled by default** (cpu, meminfo,
diskstats, filesystem, netdev, loadavg, and the rest of the safe core),
**disabled by default** (perf, systemd, interrupts, mountstats, tcpstat and
other high-cardinality or expensive sources the README explicitly warns to
enable one at a time and watch `scrape_duration_seconds`), and **deprecated**
(ntp, runit, supervisord — slated for removal)[^2]. Per-OS support varies widely:
most collectors are Linux-only, with a smaller portable set on Darwin and the
BSDs, since the whole design leans on procfs/sysfs semantics.

Two escape hatches matter. The **textfile collector**
(`--collector.textfile.directory`) reads `*.prom` files off local disk, letting
cron jobs and custom scripts expose machine-tied metrics without running their
own exporter — the standard pattern for batch-job completion times and static
role labels. Scrape-time filtering via `collect[]` / `exclude[]` URL params lets
different Prometheus servers pull different metric subsets from the same target.

## Production Notes

**The filesystem collector blocks on hung mounts.** Its `statfs` calls can hang
on an unresponsive NFS or network mount, stalling the whole scrape until
`scrape_timeout`. Exclude network/virtual mount points with
`--collector.filesystem.mount-points-exclude` and `--collector.filesystem.fs-types-exclude`.
This is the single most common operational surprise.

**Containers monitor the container, not the host, unless you break isolation.**
Running in Docker requires `--net=host --pid=host`, a read-only bind of `/` into
the container, and `--path.rootfs=/host` so the collectors prefix host paths
correctly[^2]. Miss any of these and your "host" metrics silently describe the
container namespace. Some collectors need extra capabilities (`--cap-add SYS_TIME`
for `timex`; `kernel.perf_event_paranoid` tuning for `perf`).

**High-cardinality collectors can overwhelm your TSDB.** `perf`, `systemd`,
`interrupts`, `mountstats`, and `tcpstat` are disabled by default for good
reason. Enabling them on a busy host can multiply series count and push scrape
duration past the interval. The README's advice — enable one at a time, test off
production, watch `scrape_duration_seconds` and `scrape_samples_post_metric_relabeling`
— is not optional.

**No auth/TLS by default.** The metrics endpoint is plaintext HTTP with no
authentication out of the box. TLS and basic auth are available via a
`--web.config.file` (the shared Prometheus exporter-toolkit), but you must
configure it; many deployments simply firewall port 9100 to the Prometheus
server instead.

**Upgrades have broken things twice in memorable ways.** The 0.16 release
renamed nearly every metric to follow Prometheus naming conventions (base units,
`_total` suffixes — e.g. `node_cpu` became `node_cpu_seconds_total`), breaking
essentially every dashboard and alert rule of the era[^3]. The 1.0 release
changed collector selection from a single comma-separated `--collectors.enabled`
flag to per-collector boolean flags, so old init scripts stop working on
upgrade. Pin the version and read release notes before bumping.

## When to Use / When Not

**Use when:**
- You run Prometheus and need standard host/hardware metrics from Linux or BSD.
- You want a stable, unopinionated agent that reads kernel counters and nothing more.
- You need machine-tied custom metrics via the textfile collector.

**Avoid when:**
- You're on Windows (use windows_exporter) or need GPU metrics (dcgm-exporter).
- You want container-level or per-cgroup metrics (use cAdvisor).
- You want application/service metrics — instrument the app or use a purpose-built exporter.
- You want an all-in-one agent with its own storage, dashboards, and alerting.

## Alternatives

- prometheus-community/windows_exporter — use instead when the host is Windows; node_exporter is *NIX-only.
- google/cadvisor — use instead when you need per-container/cgroup resource metrics rather than whole-host.
- NVIDIA/dcgm-exporter — use alongside/instead for NVIDIA GPU utilization and health metrics.
- netdata/netdata — use instead when you want a self-contained host agent with built-in storage and dashboards, not a Prometheus scrape target.
- influxdata/telegraf — use instead when you want one pluggable agent covering host plus many services, pushing to InfluxDB/others (also speaks Prometheus).

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-04 | First commit; early Prometheus-era host exporter[^1]. |
| 0.16.0 | 2018-05 | Metric-naming overhaul to Prometheus conventions — mass breaking change[^3]. |
| 0.18.0 | 2019-05 | Collector expansion; late-0.x stabilization. |
| 1.0.0 | 2020-06 | First stable release; per-collector flag scheme replaces `--collectors.enabled`[^4]. |
| 1.6.0 | 2023 | Continued collector additions; TLS/auth via web config maturing. |
| 1.8.0 | 2024-04 | Further collectors and fixes on the 1.x line. |

## References

[^1]: prometheus/node_exporter — repository and history. https://github.com/prometheus/node_exporter
[^2]: node_exporter README — installation, Docker flags, collector tables, textfile collector. https://github.com/prometheus/node_exporter/blob/master/README.md
[^3]: Prometheus docs, "Version 0.16.0 migration" (metric renaming). https://prometheus.io/docs/guides/node-exporter/ and release notes. https://github.com/prometheus/node_exporter/releases/tag/v0.16.0
[^4]: node_exporter v1.0.0 release notes. https://github.com/prometheus/node_exporter/releases/tag/v1.0.0

## Tags

go, prometheus, monitoring, observability, metrics, host-metrics, exporter, procfs, sysfs, linux, system-metrics, pull-based
