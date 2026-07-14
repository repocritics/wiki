# elastic/kibana

> The browser front end for Elasticsearch — search, dashboards, and the operational apps (observability, security) built on top of them.

[GitHub repo](https://github.com/elastic/kibana) ·
[Official website](https://www.elastic.co/kibana) ·
[License: Elastic-2.0 / SSPL-1.0 (dual)](https://github.com/elastic/kibana/blob/main/LICENSE.txt)

## Overview

Kibana is the visualization and management layer of the Elastic Stack. It has no data store of its own: it is a Node.js server plus a React single-page app that talks to Elasticsearch over its REST API and renders the results as searches, charts, dashboards, and maps. It began in 2013 as Rashid Khan's client-side front end for Logstash/Elasticsearch log data[^1], and grew into the "K" of the ELK stack (Elasticsearch, Logstash, Kibana).

Over time Kibana stopped being just a chart tool. It is now the delivery vehicle for Elastic's whole product line — Observability (logs/metrics/APM/uptime), Security (SIEM and endpoint), Enterprise Search, Maps, Machine Learning, and Fleet/agent management all ship as Kibana apps. This is the defining tension: a large fraction of the codebase is not "dashboarding" but full applications that happen to be hosted inside Kibana's plugin platform. The result is one of the largest TypeScript monorepos in open source, and a product whose surface area far exceeds what most users touch.

The other defining fact is licensing. Kibana was Apache 2.0 until early 2021, when Elastic relicensed both Elasticsearch and Kibana under a dual Elastic License 2.0 / Server Side Public License scheme[^2]. Neither is an OSI-approved open-source license; GitHub reports the license as `NOASSERTION`. AWS responded by forking the Apache-2.0 lineage into OpenSearch and OpenSearch Dashboards. If you need a genuinely OSI-open Kibana, the fork — not this repo — is the answer.

## Getting Started

Kibana is version-locked to Elasticsearch, so the simplest path is Docker Compose with a matching pair, or Elastic's `start-local` script:

```bash
# One-command local Elasticsearch + Kibana (dev only, single node)
curl -fsSL https://elastic.co/start-local | sh
# Kibana comes up on http://localhost:5601
```

Minimal `kibana.yml` pointing at an existing cluster:

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["https://es01:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_PASSWORD}"
# Required once you use alerting, Fleet, or saved-object encryption:
xpack.encryptedSavedObjects.encryptionKey: "something_at_least_32_characters"
```

Then run `bin/kibana` (or the container). Point a browser at `:5601`, open **Dev Tools → Console**, and you can drive Elasticsearch directly:

```
GET /_cat/indices?v
POST /my-index/_search
{ "query": { "match": { "message": "error" } } }
```

## Architecture / How It Works

Kibana is a **plugin platform**, not a monolith. A small `core` (bootstrapping, HTTP server, saved-objects service, Elasticsearch client, UI settings) exposes lifecycle contracts (`setup`, `start`), and everything else — Discover, Dashboard, Lens, Maps, Security, Observability — is a plugin that consumes those contracts[^3]. Plugins have both a server-side half (Node.js) and a browser-side half (React), and declare dependencies on each other explicitly. This "New Platform" rewrite spanned the 7.x series and replaced the earlier Hapi-based legacy architecture.

Key internals an operator should understand:

- **Saved objects.** Dashboards, visualizations, index patterns (data views), rules, and Kibana's own configuration are stored as documents in system indices inside Elasticsearch (historically `.kibana`, now a family of `.kibana_*` indices), not in a separate database. Kibana owns and migrates these indices.
- **Query languages.** KQL (Kibana Query Language) is the default filter syntax; Lucene query string is still available; ES|QL, a piped query language, was added more recently as a first-class query and visualization path[^4].
- **Lens** is the drag-and-drop visualization builder that has become the default authoring surface, layered over the older Visualize/TSVB editors that remain for back-compat.
- **Reporting** (PDF/PNG export) runs a bundled headless Chromium via Puppeteer inside the Kibana server process — a heavyweight dependency that dominates the container image size and RAM footprint.
- **Alerting, Actions, and Task Manager** persist rules as saved objects and poll Elasticsearch on a schedule; Task Manager uses an Elasticsearch index as its job queue.

The whole thing is a single Node.js process (optionally clustered). Rendering is server-assisted but primarily a client-side React app; the server proxies and shapes Elasticsearch requests rather than doing heavy computation itself.

## Production Notes

**Version lockstep with Elasticsearch is strict.** Kibana refuses to start against an Elasticsearch of an older or newer *major* version, and warns on minor/patch skew[^5]. Upgrades are therefore coordinated stack-wide, not per-component.

**Saved-object migrations are the upgrade risk.** On startup after an upgrade, Kibana migrates its system indices to the new schema. On large or heavily-used deployments this can be slow, and a failed migration blocks startup. Snapshot the `.kibana*` indices before upgrading; a single misbehaving Kibana node mid-migration can wedge the others.

**It is memory-hungry.** The Node process routinely wants 1–2 GB of heap for non-trivial workloads, and the bundled Chromium for Reporting adds hundreds of MB more when active. `--max-old-space-size` tuning and giving Reporting its own capacity are common needs.

**Encryption keys must be set and stable.** Alerting, Fleet, and encrypted saved objects require `xpack.encryptedSavedObjects.encryptionKey` (and the reporting/security keys). If you don't set them, Kibana generates ephemeral keys and warns; if you rotate them without a migration, previously encrypted objects become unreadable. Set them once, in the keystore, and keep them.

**Security is on by default in 8.x+.** Older habits of running Kibana against an unauthenticated cluster no longer work out of the box; TLS and a `kibana_system` service account are expected.

**Open issue count is not decay.** The repo shows a five-figure open-issue count, which for a monorepo hosting dozens of shipped products reflects breadth and label-based triage across many teams, not abandonment — commits land daily.

## When to Use / When Not

**Use when:**
- Your data already lives in Elasticsearch and you want the native, deeply-integrated UI for it.
- You need the operational apps — APM, SIEM/Security, Observability, Fleet — as a packaged whole rather than assembling them.
- You want ad-hoc log/document exploration (Discover) with KQL/ES|QL over indexed data.
- You are already committed to the Elastic Stack and licensing terms are acceptable.

**Avoid when:**
- You need a genuinely OSI-open license — use the OpenSearch Dashboards fork instead.
- Your dashboards must span many heterogeneous data sources (SQL DBs, Prometheus, cloud metrics) — Grafana is source-agnostic by design.
- Your data isn't in Elasticsearch and you don't want to run and operate a cluster just to get a UI.
- You want something lightweight; Kibana's footprint and operational surface are large.

## Alternatives

- opensearch-project/OpenSearch-Dashboards — the Apache-2.0 fork of Kibana; use it when you need an OSI-open license and are (or can be) on OpenSearch instead of Elasticsearch.
- grafana/grafana — use it when dashboards must unify many data sources (Prometheus, SQL, cloud) rather than Elasticsearch alone.
- apache/superset — use it when your data is in SQL warehouses and you want BI-style exploration and charts.
- getredash/redash — use it when analysts want to share query-driven dashboards written in SQL.
- metabase/metabase — use it when non-technical users need self-serve question-and-dashboard building over relational sources.

## History

| Version | Date | Notes |
|---------|------|-------|
| 3.0 | 2014-04 | Browser-only dashboards; no server component[^1]. |
| 4.0 | 2015-02 | Introduced the Node.js server back end. |
| 5.0 | 2016-10 | Unified Elastic Stack versioning aligned Kibana with ES/Logstash/Beats. |
| 6.0 | 2017-11 | Index pattern rework, rolling upgrade support. |
| 7.0 | 2019-04 | New Platform plugin architecture and redesigned navigation[^3]. |
| 7.11 | 2021-02 | Relicensed from Apache 2.0 to dual Elastic License 2.0 / SSPL[^2]. |
| 8.0 | 2022-02 | Security on by default; legacy platform removed. |
| 9.0 | 2025-04 | Major release across the 9.x Elastic Stack line. |

## References

[^1]: Elastic, "The Story of Kibana" / project history. https://www.elastic.co/kibana
[^2]: Elastic, "Doubling down on open, Part II" — license change announcement, 2021-01-19. https://www.elastic.co/blog/licensing-change
[^3]: Kibana developer docs, "Kibana Platform" plugin architecture. https://www.elastic.co/docs/extend/kibana/
[^4]: Elastic, ES|QL (Elasticsearch Query Language) documentation. https://www.elastic.co/docs/explore-analyze/query-filter/languages/esql
[^5]: Kibana README, "Version Compatibility with Elasticsearch". https://github.com/elastic/kibana#version-compatibility-with-elasticsearch

## Tags

typescript, react, nodejs, elasticsearch, dashboards, observability, data-visualization, siem, log-analytics, elastic-stack, business-intelligence
