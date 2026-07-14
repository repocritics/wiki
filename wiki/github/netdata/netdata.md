# netdata/netdata

> Per-second, zero-config infrastructure monitoring that runs a self-contained agent on every node instead of shipping metrics to a central store.

[GitHub repo](https://github.com/netdata/netdata) ·
[Official website](https://www.netdata.cloud) ·
[License: GPL-3.0](https://github.com/netdata/netdata/blob/master/LICENSE)

## Overview

Netdata is a real-time monitoring agent, started by Costa Tsaousis in 2013[^1], that inverts the usual observability topology: instead of a central time-series database scraping thin exporters, each node runs a full agent that collects, stores, applies ML, evaluates alerts, and serves its own dashboard locally. Point a browser at `http://node:19999` and you get per-second charts within seconds of install, with no scrape config, no query language, and no separate visualization stack to stand up. The auto-discovery and "it just works on first run" experience is the project's real differentiator against the assemble-it-yourself Prometheus + Grafana + exporters stack.

The defining tension is open-core. The agent itself is GPL-3.0 and genuinely self-hostable, but the bundled dashboard ("Netdata UI") ships under a proprietary license (NCUL1) and the multi-node, RBAC, and long-horizon-aggregation features live in the commercial **Netdata Cloud** SaaS[^2]. The product is nudged heavily toward connecting agents to Cloud. You can run fully local and offline, but you are running against the grain of where new features land, and the on-agent dashboard has become a thinner experience than the Cloud-connected one over time.

The second surprise is the language story. GitHub reports the repo as **Go**, but that reflects the large `go.d.plugin` collector tree; the performance-critical core — daemon, storage engine, streaming, ML, web server — is C[^3]. Netdata is a CNCF member project and one of the most-starred entries in the CNCF landscape.

## Getting Started

```bash
# One-line kickstart installer (Linux/macOS/FreeBSD) — detects distro,
# installs native package or builds from source, sets up auto-updates.
wget -O /tmp/netdata-kickstart.sh https://get.netdata.cloud/kickstart.sh \
  && sh /tmp/netdata-kickstart.sh

# Or Docker:
docker run -d --name=netdata -p 19999:19999 \
  --cap-add SYS_PTRACE --security-opt apparmor=unconfined \
  -v netdataconfig:/etc/netdata \
  -v netdatalib:/var/lib/netdata \
  -v /proc:/host/proc:ro -v /sys:/host/sys:ro \
  netdata/netdata
```

The dashboard is immediately live at `http://localhost:19999`. Most metrics need no configuration; per-collector settings live under `/etc/netdata/` and are edited with the bundled `edit-config` script (which copies stock defaults out of `/usr/lib/netdata/conf.d/`).

## Architecture / How It Works

Each agent runs a nine-stage pipeline in-process: **collect, store, learn, detect, check, stream, archive, query, score**.

- **Collect.** Internal C plugins handle high-frequency host metrics (`proc.plugin`, `cgroups.plugin`, `ebpf.plugin`, `apps.plugin`). Application/service metrics are mostly `go.d.plugin` (Go) modules — 800+ integrations covering nginx, Postgres, Redis, MongoDB, and so on. Legacy `python.d.plugin` collectors still exist but are being retired in favor of Go equivalents.
- **Store.** The `dbengine` time-series database keeps data in memory-mapped files at roughly 0.5 bytes per sample, with **tiered retention** — a high-resolution tier (per-second), plus down-sampled tiers (per-minute, per-hour) so you keep long history cheaply without holding every second forever.
- **Learn / Detect.** Unsupervised ML (k-means-based) trains models per metric on rolling windows at the edge and flags anomalies inline; an "anomaly bit" travels with each sample, so anomaly rates are queryable like any other metric.
- **Check.** A health engine evaluates hundreds of pre-shipped alert definitions and dispatches to email, Slack, Telegram, PagerDuty, Discord, Teams, and others.
- **Stream / Query.** Agents can **stream** to a "Parent" agent for centralization and longer retention (the Parent-Child model), and everything is exposed over an HTTP query API that the auto-generated dashboards consume. Archiving exports to Prometheus, InfluxDB, Graphite, OpenTSDB, and similar.

Dashboards are generated automatically from a data model (labeled dimensions/instances) rather than hand-built, which is why there is no query language to learn — and also why deep custom analytics are weaker than a PromQL/Grafana setup.

## Production Notes

- **The agent has no authentication on port 19999.** Anyone who can reach it sees every metric on the box. Netdata assumes you firewall it, bind it to localhost, or front it with a reverse proxy + auth. Exposing `:19999` to the internet is a common and serious misconfiguration.
- **The bundled UI is not open source.** It ships under NCUL1, and full dashboard functionality (and the newer UI) leans on assets served from Netdata's CDN and, for multi-node views, on Cloud. Fully air-gapped deployments work but lose parts of the experience; verify your offline story before committing.
- **Anonymous telemetry is on by default.** The agent phones home usage statistics and registers with a public registry unless you opt out at install (`--disable-telemetry`) or afterward.
- **Memory and disk scale with cardinality.** Per-second collection across thousands of metrics, high container churn (short-lived cgroups), and multiple retention tiers all inflate `dbengine` memory and disk. On constrained nodes, tune `update_every`, cap `dbengine` tier sizes, or disable unneeded collectors.
- **Parents need sizing.** Centralizing many children onto a Parent concentrates CPU, memory, and disk there; a Parent is effectively a monitoring server and should be provisioned like one.
- **v1 → v2 shifted the default experience** toward the Cloud-connected React UI. Teams that relied on the older fully-local agent dashboard should confirm the current on-agent UI still meets their needs after upgrades.

## When to Use / When Not

**Use when:**
- You want turnkey, per-second host and container visibility with near-zero setup.
- You run many similar nodes and value auto-discovery over hand-written scrape configs.
- You want built-in anomaly detection and alerting without wiring a separate ML or rules stack.
- You need low-overhead monitoring on edge/IoT or resource-constrained hosts.

**Avoid when:**
- You need a rich query language and custom analytics — Prometheus + Grafana is more flexible.
- You require a single centralized long-term metrics store as the system of record (Netdata's model is per-node-first; centralization is via Parents/Cloud).
- Open-source purity is a hard requirement and a proprietary bundled UI / Cloud nudge is unacceptable.
- You must expose dashboards publicly and don't want to build the auth/proxy layer yourself.

## Alternatives

- prometheus/prometheus — use instead when you want pull-based metrics with PromQL and a central query engine, and are willing to configure exporters yourself.
- grafana/grafana — use instead (as the viz layer over a TSDB) when dashboards across many data sources matter more than an all-in-one agent.
- influxdata/telegraf — use instead when you want a pluggable collection agent that ships to your own database rather than storing and rendering locally.
- prometheus/node_exporter — use instead when you only need host metrics feeding an existing Prometheus stack.
- zabbix/zabbix — use instead when you want a mature, server-driven monitoring system with template-based config and built-in alerting.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013 | Project started by Costa Tsaousis; repo created on GitHub[^1]. |
| 1.x | 2016–2024 | Long-lived v1 line: `dbengine` tiered storage, streaming Parents, edge ML anomaly detection, Netdata Cloud, 800+ collectors. |
| 2.0 | 2024 | Major release centered on the new Cloud-connected UI and dashboard model[^4]. |

## References

[^1]: Netdata README, origin story ("In 2013, at the company where Costa Tsaousis was COO…"). https://github.com/netdata/netdata
[^2]: Netdata ecosystem table — Agent (GPL v3+), Cloud (commercial), UI (NCUL1). https://github.com/netdata/netdata#netdata-ecosystem and https://app.netdata.cloud/LICENSE.txt
[^3]: Netdata agent architecture / distributed observability pipeline. https://learn.netdata.cloud/docs/netdata-agent/
[^4]: Netdata releases. https://github.com/netdata/netdata/releases

## Tags

go, c, monitoring, observability, metrics, time-series, anomaly-detection, cncf, devops, kubernetes, prometheus, agent
