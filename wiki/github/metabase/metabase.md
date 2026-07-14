# metabase/metabase

> Open-source, self-service business intelligence — a GUI query builder and dashboard layer that sits in front of your existing databases.

[GitHub repo](https://github.com/metabase/metabase) ·
[Official website](https://metabase.com) ·
[License: AGPL-3.0 (OSS edition) + commercial](https://github.com/metabase/metabase/blob/master/LICENSE.txt)

## Overview

Metabase is a business-intelligence application that connects to your existing SQL databases and warehouses and lets non-technical users build charts, dashboards, and alerts without writing SQL[^1]. It was started in 2015 by a team spun out of the Expa startup studio and is now developed by Metabase, Inc. The backend is Clojure; the frontend is React/TypeScript. It ships as a single runnable JAR (or Docker image) with an embedded H2 application database, which is the source of its "set up in five minutes" reputation and, as noted below, its most common production footgun.

The defining tension is **open core**. The repository holds two editions under one tree: the Open Source edition under AGPL-3.0, and the paid Pro/Enterprise editions under the proprietary Metabase Commercial Software License[^2]. Features such as row/column-level data sandboxing, SSO/SAML, official serialization for environment promotion, white-label embedding, and audit logging live behind the commercial license. The GitHub `license` field resolves to `NOASSERTION` precisely because the `LICENSE.txt` mixes both — the OSS half is genuinely AGPL, but assume any advanced governance/embedding feature may be paywalled until you check.

Metabase is aimed at teams that want self-service analytics without standing up a warehouse-plus-Looker stack. It does not store or transform your data by default — it is a query and presentation layer over databases you already run. With ~48k stars and multi-daily commit activity as of 2026, it is one of the most actively maintained open-source BI tools.

## Getting Started

The fastest path is Docker or the JAR. Both start with the embedded H2 app DB (fine for evaluation only):

```bash
# Docker
docker run -d -p 3000:3000 --name metabase metabase/metabase

# or the JAR (needs a JRE, Java 21 recommended)
java -jar metabase.jar
```

For any real deployment, point the application database at Postgres via environment variables before first launch:

```bash
docker run -d -p 3000:3000 \
  -e MB_DB_TYPE=postgres \
  -e MB_DB_DBNAME=metabaseapp \
  -e MB_DB_PORT=5432 \
  -e MB_DB_USER=metabase \
  -e MB_DB_PASS=secret \
  -e MB_DB_HOST=postgres.internal \
  --name metabase metabase/metabase
```

Then open `http://localhost:3000`, complete the setup wizard, and add a connection to the *data* database you want to analyze (separate from the app DB above).

## Architecture / How It Works

Metabase separates two databases that new operators frequently conflate:

1. **Application database** — stores Metabase's own state: users, permissions, saved questions, dashboards, collections. Defaults to embedded H2; should be Postgres or MySQL in production. Schema is managed by Liquibase migrations that run automatically on startup.
2. **Data databases** — the sources you connect and query (Postgres, MySQL, BigQuery, Snowflake, Redshift, ClickHouse via community driver, etc.). Metabase never copies this data wholesale; it queries live.

The query engine is built around **MBQL** (Metabase Query Language), an internal S-expression-style representation. The GUI query builder produces MBQL; the query processor then compiles MBQL to native SQL (or the driver's dialect) per target database[^3]. Drivers are Clojure multimethods — this is how the same "question" runs against both Postgres and BigQuery, and how community drivers plug in.

On connecting a database, Metabase runs **sync, scan, and fingerprint** jobs that walk the schema, sample column values, and infer field types/semantic types. This metadata powers the GUI (dropdown filters, auto-binning, X-rays). On very large warehouses these background jobs are non-trivial and are a common source of load complaints.

The backend uses Ring/Compojure for HTTP and the in-house Toucan ORM over the app DB. The frontend is React + Redux + TypeScript, served as a SPA from the same JAR. There is no separate services architecture — Metabase is a monolith by design, which keeps operation simple but means horizontal scaling is "run more identical JVMs behind a load balancer, all pointed at one shared app DB."

## Production Notes

**The H2 app-database trap.** The default embedded H2 database is not safe for production: it corrupts under crashes, cannot be backed up cleanly while running, and blocks running multiple app instances. Migrate to Postgres/MySQL *before* accumulating content — the built-in migration (`load-from-h2`) is one-way and best done early. This single misconfiguration accounts for a large share of "I lost all my dashboards" reports.

**Upgrades run migrations on startup and are not reversible.** Liquibase applies schema changes when a newer version boots against your app DB. Downgrades are unsupported; always snapshot the app DB before upgrading. Skipping several major versions at once mostly works but is riskier — step through if you are far behind.

**JVM memory.** Metabase is a JVM app; large result sets, CSV/XLSX exports, and dashboard subscriptions are the usual OOM triggers. Tune with `JAVA_OPTS`/`-Xmx`; give it real heap (2–4 GB+ for busy instances). Exports stream to memory in older versions.

**Query performance is your warehouse's problem, not Metabase's.** It pushes work down to the source DB. Slow dashboards usually mean unindexed columns or an under-provisioned warehouse. Mitigations inside Metabase: query result caching, model persistence (materializing models to writeback tables), and limiting auto-refresh dashboards.

**Embedding and licensing.** Static (signed-JWT) embedding exists in OSS but keeps Metabase branding; removing branding, interactive embedding, and multi-tenant data sandboxing are commercial features. Because the OSS edition is AGPL, embedding it in a proprietary product has copyleft implications — teams that want to avoid that typically buy a commercial license[^2].

**Version promotion.** Moving content dev→staging→prod cleanly (serialization/import-export) is an enterprise feature; OSS users often resort to app-DB copies, which drags along environment-specific IDs.

## When to Use / When Not

**Use when:**
- You want non-analysts to self-serve charts and dashboards over existing SQL databases.
- You want to self-host BI on a single JAR/container without a heavy data-stack buildout.
- You need email/Slack alerts and scheduled dashboard subscriptions out of the box.
- Your governance needs are modest, or your budget covers the Pro/Enterprise tier.

**Avoid when:**
- You need a version-controlled, code-defined semantic layer (LookML-style) as the source of truth — Metabase's model layer is GUI-first.
- You require fine-grained governance (row-level sandboxing, SSO, audit) but cannot pay for the commercial edition.
- Your analytics require heavy in-tool transformation/ETL — Metabase queries live and is not an ELT engine (its Data Studio transforms are limited compared to dbt).
- AGPL copyleft is unacceptable and you cannot license the commercial edition.

## Alternatives

- apache/superset — Apache-2.0, permissive-licensed BI with deeper chart customization; more operationally involved (Python/Celery/Redis stack). Use when you want no commercial-license strings and can run more infrastructure.
- getredash/redash — SQL-first query-and-visualize tool. Use when your users write SQL and you want lighter dashboarding rather than a GUI query builder.
- grafana/grafana — time-series and observability dashboards first. Use for metrics/monitoring rather than ad-hoc business questions over relational data.
- evidence-dev/evidence — code/Markdown-defined BI, git-versioned. Use when you want analytics-as-code instead of a point-and-click tool.
- Looker (Google, closed-source) — governed semantic layer via LookML. Use when a versioned, centrally-modeled metrics layer matters more than self-hosting.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-10 | First public open-source release[^1]. |
| 1.0 (0.32/1.32) | 2019-10 | 1.0 milestone; Enterprise edition introduced (open-core split)[^2]. |
| 0.41 | 2021-10 | Models, redesigned query builder era. |
| 0.46 | 2023-05 | Toucan 2 ORM, performance/permissions rework. |
| 0.50 | 2024 | Metabot/AI and analytics improvements; new major-version cadence. |

## References

[^1]: Metabase — official site and product overview. https://www.metabase.com
[^2]: Metabase source and licensing (OSS AGPL edition + commercial editions), `LICENSE.txt`. https://github.com/metabase/metabase/blob/master/LICENSE.txt
[^3]: Metabase developer documentation — query processor, drivers, and MBQL. https://www.metabase.com/docs/latest/developers-guide/start

## Tags

clojure, business-intelligence, analytics, dashboard, sql-editor, data-visualization, self-hosted, open-core, bi-tool, embedded-analytics, jvm
