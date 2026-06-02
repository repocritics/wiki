# apache/superset

Apache Superset — an OSS data exploration and visualization platform. Tableau-style BI dashboards over your existing SQL databases.

## What it is

A web-based BI tool that connects to your SQL data warehouse / database (Postgres, MySQL, BigQuery, Snowflake, Redshift, Trino, Druid, ClickHouse, etc.), exposes a SQL editor and chart builder, and lets users compose dashboards from saved charts. Originated at Airbnb, donated to the Apache Software Foundation. Now widely used as the OSS alternative to Tableau / Looker for teams that want self-hosting and SQL-native exploration.

## Key features

- 50+ chart types built on ECharts and Plotly.
- SQL Lab — IDE-style query editor with autocomplete, tabs, saved queries.
- Dashboard builder with filters, cross-filters, drilldowns.
- Connector ecosystem covering most SQL warehouses + some NoSQL.
- Row-level security, dashboard-level permissions, role-based access.
- Caching layer for query results.
- Apache 2.0 licensed.

## Tech stack

- Python (Flask) on the backend.
- TypeScript / React on the frontend.
- Celery for async query execution.
- Postgres / MySQL for metadata storage.

## When to reach for it

- You want self-hosted BI for an existing SQL warehouse.
- You're a team that wants SQL-native exploration with charting on top.
- You need ASF-governed library stability + community.

## When *not* to reach for it

- Your data isn't SQL-shaped — Superset is SQL-first.
- You want polished commercial BI — Tableau / Looker / Mode have more polish (and price tag).
- You're allergic to operational complexity — Superset needs Postgres + Celery + Redis to run well.

## Maturity signal

73k stars, 17k forks, Apache 2.0, actively maintained. 9+ years; ASF-graduated project since 2021. The 1,300 open-issues count is moderate.

## Alternatives

- Metabase — use when you want simpler, less customizable BI.
- Redash — comparable predecessor; effectively absorbed by Databricks.
- Cube + custom UI — use when you want a semantic-layer rather than dashboard-first BI.

## Tags

apache-license, business-intelligence, data-visualization, sql, python, typescript, react, flask, self-hosted, data-analytics
