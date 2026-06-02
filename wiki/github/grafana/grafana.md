# grafana/grafana

The composable observability + dashboard platform — visualize metrics, logs, and traces from many sources. The de facto monitoring dashboard.

## What it is

A TypeScript + Go application that builds dashboards over time-series and log data sources: Prometheus, Loki, Elasticsearch, InfluxDB, Postgres, MySQL, Tempo (tracing), and many more. Built at Raintank (now Grafana Labs); the open-source flagship of Grafana Labs' commercial-OSS strategy. Alerting, anomaly detection, and a vast plugin ecosystem layered on top. AGPL-3.0 licensed.

## Key features

- 100+ data-source plugins covering metrics, logs, traces, and tabular databases.
- Dashboard composition with hundreds of visualization types.
- Alerting via Grafana Alertmanager or Prometheus Alertmanager.
- Loki (logs) + Tempo (traces) + Mimir (metrics) form the Grafana-native observability stack.
- LLM-augmented features in newer versions (Grafana AI Assistant).
- Self-host (Docker, OSS install) or use Grafana Cloud.
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary on the frontend.
- Go on the backend.
- Distributed via Docker, OS packages, OSS binary releases.

## When to reach for it

- You're standing up monitoring / observability and want a polished dashboard layer.
- You have a Prometheus + Loki + Tempo stack and want the native Grafana experience.
- You need flexible visualization across heterogeneous data sources.

## When *not* to reach for it

- You're allergic to AGPL — verify implications for SaaS hosting (Grafana Labs sells the managed version).
- You want simpler — single-purpose dashboard tools (Kibana for ES, NetData for system metrics) are lighter.

## Maturity signal

74k stars, 14k forks, AGPL-3.0, actively maintained under Grafana Labs. 12+ years. Open-issues count of 3,615 reflects the plugin + integration breadth.

## Alternatives

- Datadog / New Relic — commercial SaaS.
- Kibana — Elasticsearch-flavored.
- VictoriaMetrics + custom UI.
- Splunk (commercial) — for log-heavy workflows.

## Tags

typescript, golang, observability, monitoring, dashboard, data-visualization, prometheus, agpl, self-hosted, business-intelligence
