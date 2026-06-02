# louislam/uptime-kuma

A self-hosted uptime monitoring tool — pings, status pages, and alerts in one polished Vue + Socket.IO app.

## What it is

A JavaScript monitoring tool that periodically checks HTTP endpoints, TCP ports, DNS, ping, and other targets, then displays status + alerts in a real-time dashboard. Lightweight enough for a Raspberry Pi; Docker-first deployment. Includes a public status-page feature for incident communication. MIT-licensed.

## Key features

- Monitor types: HTTP(S), TCP port, ping, DNS, Docker container, gRPC, MQTT, Kafka, Steam game server, more.
- Real-time dashboard with WebSocket-driven updates.
- Public status pages with custom branding.
- Notification channels: Discord, Slack, Telegram, Webhook, email, ~80 others.
- 2FA, multi-user support.
- Lightweight: runs comfortably on a small Pi / VPS.
- MIT-licensed.

## Tech stack

- JavaScript / Vue.js primary.
- Socket.IO for real-time updates.
- SQLite (or MariaDB) for persistence.

## When to reach for it

- You want a polished self-hosted uptime monitor for personal / homelab / small-team use.
- You want public status pages without Statuspage.io's pricing.

## When *not* to reach for it

- You're at scale — Uptime Kuma is single-node; large-scale monitoring wants Prometheus + Alertmanager.
- You need enterprise SLA support.

## Maturity signal

88k stars, 8k forks, MIT, actively maintained. Open-issues count of 762 tracks the breadth of monitor types + notification channels.

## Alternatives

- Statping-ng — alternative self-hosted status / uptime tool.
- Prometheus + Alertmanager + Grafana — for scale.
- Statuspage.io, Better Uptime — commercial managed.

## Tags

javascript, vue, monitoring, uptime, self-hosted, mit-license, websocket, dashboard
