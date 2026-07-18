# prometheus/alertmanager

> The alert delivery layer of the Prometheus stack — deduplicates, groups,
> routes, silences, and inhibits alerts before they reach a human.

[GitHub repo](https://github.com/prometheus/alertmanager) ·
[Official website](https://prometheus.io) ·
[License: Apache-2.0](https://github.com/prometheus/alertmanager/blob/main/LICENSE)

## Overview

Alertmanager receives alerts pushed by Prometheus servers (or anything that
speaks its HTTP API) and decides *who* gets notified, *when*, and *how
often*. Prometheus evaluates alerting rules; Alertmanager owns everything
after "the rule fired": dedup across HA Prometheus pairs, grouping related
alerts into one notification, a routing tree mapping label sets to receivers
(email, PagerDuty, OpsGenie, Slack, webhooks, more), silences, and
inhibition (mute warnings while the matching critical fires)[^1].

The project dates to 2013; the code that ships today is a ground-up rewrite
first released as 0.1.0 in February 2016[^2]. A decade later it is still
versioned 0.x — the team has never declared a 1.0 — yet it is among the
most widely deployed alerting infrastructure anywhere: every
kube-prometheus-stack install ships one, and Grafana, Mimir, and Cortex all
embed forks of it. Its ~8.5k stars understate the footprint; nobody stars
pager plumbing. Maintenance is active (0.33.1 released 2026-07).

The defining design tension: alerts are **ephemeral, repeatedly-resent
state**, not durable messages. Firing alerts live only in memory; a
restarted Alertmanager knows nothing until Prometheus resends. That keeps
the system simple and replicable, but it is not a queue, not an audit log,
and not an incident tracker — teams expecting any of those get surprised.

## Getting Started

```bash
docker run --name alertmanager -d -p 127.0.0.1:9093:9093 \
  quay.io/prometheus/alertmanager
```

Minimal `alertmanager.yml`:

```yaml
route:
  receiver: team-pager
  group_by: ['alertname', 'cluster']
  group_wait: 30s        # wait to batch alerts arriving together
  group_interval: 5m     # min gap between notifications for a group
  repeat_interval: 3h    # re-notify while still firing
receivers:
  - name: team-pager
    pagerduty_configs:
      - routing_key: <key>
```

Point Prometheus at it via the `alerting.alertmanagers` block in
`prometheus.yml`. `amtool`, bundled with every release, drives the API from
the CLI; `amtool config routes test` dry-runs the routing tree against a
label set — the sanest way to verify routing changes before deploy.

## Architecture / How It Works

Incoming alerts land in an in-memory store. A **dispatcher** walks the
routing tree — label matchers where child routes inherit and override
parent settings — and assigns each alert to an **aggregation group** keyed
by the route's `group_by` labels. Each group runs the timing state machine:
`group_wait` before the first notification, `group_interval` between
updates, `repeat_interval` for re-notification. On flush, a group passes
through the **notification pipeline**: inhibition → silence → cluster wait
→ dedup against the **notification log** (nflog) → send with retries.

High availability is gossip-based (hashicorp/memberlist, adopted in
0.15.0[^3]): peers replicate silences and the nflog, not alerts themselves.
Prometheus sends every alert to **every** Alertmanager; each peer delays
sending by its cluster position, and the gossiped nflog lets later peers
see that peer 0 already notified. This is at-least-once delivery —
duplicates during partitions are by design: a duplicate page beats a missed
one.

Silences and the nflog are snapshotted to local disk (`--storage.path`);
alert state is not. The web UI is written in Elm (hence the Node.js build
dependency). The v2 API is OpenAPI-generated; v1 was removed in 0.27.0[^4].

## Production Notes

- **Never load-balance Prometheus → Alertmanager traffic.** Each Prometheus
  must send to all peers directly; a load balancer breaks the
  dedup-by-nflog model and produces missed or duplicate pages[^5].
- Clustering needs the cluster port open on **both TCP and UDP** (0.15+);
  TCP-only firewalls cause silent gossip degradation and double pages[^5].
- Silences propagate by gossip; a notification can still escape before a
  fresh silence reaches all peers. HA is eventually consistent.
- `repeat_interval` is evaluated at `group_interval` boundaries, so it is
  effectively rounded up to a multiple of `group_interval`.
- Inhibition footgun: if every label in an `equal` list is missing from
  *both* source and target alerts, the rule still matches — the README
  itself carries a CAUTION about this[^5].
- `group_by: [...]` (aggregate by all labels) disables grouping entirely;
  with real alert volume this floods receivers.
- Restarts drop firing-alert state. Prometheus resends on an interval so
  the gap is bounded, but a resolved notification can be lost if the alert
  resolves while Alertmanager is down.
- Upgrade edges: 0.27.0 removed API v1[^4] — anything still posting to
  `/api/v1/alerts` breaks. 0.27 also began the UTF-8 matcher-syntax
  migration; check logs for matcher warnings before upgrading past it.
- There is no multi-tenancy: one config, one routing tree, one silence
  namespace. Isolation is by convention (route subtrees) or by running
  Mimir/Cortex, which embed a multi-tenant fork.

## When to Use / When Not

**Use when:**
- You run Prometheus (or Mimir/Thanos/VictoriaMetrics) — it is the default
  and best-integrated alert router for that ecosystem.
- You want grouping, silencing, and routing as declarative config in
  version control rather than clicking around a SaaS.
- You need HA paging that survives node loss with zero external
  dependencies (no database, no broker).

**Avoid when:**
- You need on-call schedules, escalation policies, or incident timelines —
  Alertmanager stops at delivery; pair it with PagerDuty, GoAlert, etc.
- You need a durable audit trail of alerts; state is ephemeral by design.
- You need per-tenant isolation without running Mimir/Cortex.
- Your alerts originate outside a Prometheus-style resend loop — fire-once
  events from arbitrary systems fit a queue-backed pipeline better.

## Alternatives

- grafana/grafana — Grafana Alerting embeds an Alertmanager fork; use it
  when you want rules, routing, and dashboards managed in one UI.
- grafana/mimir — use for horizontally scaled, multi-tenant Prometheus
  including a multi-tenant Alertmanager.
- target/goalert — use for the layer Alertmanager lacks: self-hosted
  on-call schedules and escalation chains.
- keephq/keep — use when aggregating alerts from many non-Prometheus
  sources with workflow automation on top.
- moira-alert/moira — use in Graphite-centric stacks wanting a
  self-contained alerting service with built-in escalation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.x | 2013–2015 | Original prototype alongside early Prometheus. |
| 0.1.0 | 2016-02 | First release of the full rewrite: routing tree, grouping model[^2]. |
| 0.15.0 | 2018-06 | HA clustering on hashicorp/memberlist gossip; TCP+UDP required[^3]. |
| 0.16.0 | 2019-01 | OpenAPI-generated API v2; v1 deprecated. |
| 0.22.0 | 2021-05 | New matcher syntax including negative matchers; mute time intervals. |
| 0.24.0 | 2022-03 | `active_time_intervals`; `mute_time_intervals` generalized to `time_intervals`. |
| 0.27.0 | 2024-02 | API v1 removed; UTF-8 matcher support begins[^4]. |
| 0.28.0 | 2025-01 | Jira receiver added. |
| 0.33.1 | 2026-07 | Current release line. |

## References

[^1]: Alertmanager documentation. https://prometheus.io/docs/alerting/latest/alertmanager/
[^2]: Tag `0.1.0`, 2016-02-23. https://github.com/prometheus/alertmanager/releases
[^3]: Release v0.15.0, 2018-06-22 (memberlist-based HA). https://github.com/prometheus/alertmanager/releases/tag/v0.15.0
[^4]: Release v0.27.0, 2024-02-28 (API v1 removal). https://github.com/prometheus/alertmanager/releases/tag/v0.27.0
[^5]: Alertmanager README — High Availability and configuration example. https://github.com/prometheus/alertmanager#high-availability

## Tags

go, alerting, monitoring, observability, notifications, deduplication, alert-routing, high-availability, incident-response, prometheus-ecosystem, sre
