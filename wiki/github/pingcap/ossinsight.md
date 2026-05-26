# pingcap/ossinsight

> Open source software insight dashboard powered by GH Archive + TiDB. The closest existing public project to RepoCritics — and a useful contrast.

[GitHub repo](https://github.com/pingcap/ossinsight) ·
[Official website](https://ossinsight.io) ·
[License: Apache-2.0](https://github.com/pingcap/ossinsight/blob/main/LICENSE)

## Overview

OSSInsight is a dashboard and exploration tool built by PingCAP (the TiDB company) to surface trends in GitHub open source activity. It ingests the public [GH Archive](https://www.gharchive.org/) event stream into a TiDB Cloud cluster and exposes the result as a website, an API, and a collection of ready-made queries[^1]. The project is also a marketing surface for TiDB — many of the headline features showcase TiDB's HTAP (hybrid transactional/analytical) query capabilities at multi-billion-row scale.

This wiki page exists in part to make the distinction with [RepoCritics](https://repocritics.com) explicit. They are adjacent but not the same thing:

| Aspect | OSSInsight | RepoCritics |
|--------|------------|-------------|
| Primary data source | GH Archive (public events: stars, PRs, issues, pushes) | GitHub REST + GraphQL APIs, plus structured wiki content |
| Primary unit of interest | Repos and contributors, aggregated globally | Individual repos with critical, opinionated, human/AI editorial layers |
| Output | Numerical dashboards, trend charts, leaderboards | Pre-processed critic-style summaries, production caveats, alternatives |
| Editorial layer | None (data only) | 4-layer review process: AI auto-review, trusted editors, community LGTM, human escalation |
| Cross-reference graph | Implicit (cooccurrence, collaboration) | Explicit (alternatives, deprecation pointers) |
| License | Site/code Apache-2.0; data flows from GH Archive (CC0-equivalent) | Wiki content CC-BY-SA 4.0 |
| Target user | Analyst, investor, dev-rel doing market research | Developer or AI agent choosing/evaluating a specific repo |

In other words: OSSInsight tells you that React got 31,000 stars last quarter. RepoCritics tells you when you should not use React. They can be complementary; they are not substitutes.

## Getting Started

OSSInsight is primarily consumed as a website at [ossinsight.io](https://ossinsight.io). The repo also provides an API and the option to self-host against your own TiDB cluster.

```bash
# Local development
git clone https://github.com/pingcap/ossinsight
cd ossinsight
pnpm install
pnpm dev
```

Querying the public API:

```bash
curl 'https://api.ossinsight.io/v1/repos/facebook/react/'
curl 'https://api.ossinsight.io/v1/repos/facebook/react/stars/'
```

Self-hosting requires a TiDB Cloud cluster (or local TiDB) with the GH Archive ETL job loaded. The repo provides SQL DDL, Docker compose for the website, and the loader scripts; full data load is multi-terabyte and is not feasible on a laptop.

## Architecture / How It Works

OSSInsight has four layers[^1][^2]:

1. **Ingest** — GH Archive publishes hourly JSON gzips of every public GitHub event. OSSInsight has a loader that converts these into rows in TiDB tables (`github_events`, with sub-types and partitioning by date).
2. **Storage** — TiDB Cloud (the managed offering of [pingcap/tidb](https://github.com/pingcap/tidb)). TiDB is HTAP: row store for OLTP, columnar store (TiFlash) replicated from the row store for OLAP. Most of OSSInsight's dashboards are columnar scans.
3. **Query API** — A Node.js (NestJS) layer that exposes named queries as REST endpoints. Each endpoint has caching, rate limiting, and a SQL template.
4. **Frontend** — Docusaurus + custom React, with embedded chart components (ECharts). The "Explore" page exposes a SQL-with-LLM interface that translates natural language to SQL against the OSSInsight schema.

The architecture is interesting for a wiki reader because it is one of the cleanest public demonstrations of "ETL a public event firehose into HTAP storage and expose dashboards." If you are building a similar analytics product on top of a public dataset (Hacker News firehose, PyPI download logs, Crates.io, etc.), OSSInsight is the closest public reference implementation.

## Production Notes

**Data freshness.** GH Archive publishes hourly; OSSInsight's typical end-to-end lag is 1–2 hours. Anything more real-time requires the GitHub Events API directly (and the rate limits that come with it).

**GH Archive completeness.** GH Archive covers *public* events. Private repo activity, deleted-account activity, and force-pushed-away commits never appear. Star counts are subject to retroactive cleanup when GitHub purges spam accounts; historical numbers can drift downward without notice[^3].

**API rate limits.** The public API at `api.ossinsight.io` rate-limits unauthenticated requests. Bulk consumers should self-host or use the TiDB Cloud public dataset directly.

**TiDB lock-in.** The queries are tuned for TiDB. Porting to Postgres / ClickHouse / BigQuery is possible but non-trivial — several rely on TiDB-specific behaviors around index merging and TiFlash. The benchmarking blog posts (e.g., "100 billion rows")[^4] are the clearest description of the tuning.

**Star count as a proxy.** Most of OSSInsight's headline metrics are derived from stars, forks, and PR counts. These are well known to be gameable[^5] and conflate "trending in the West English-speaking developer community" with "actually used". Treat dashboards as directional, not authoritative.

## When to Use / When Not

**Use when:**
- You need cross-repo trend analysis: "what frameworks are gaining stars this quarter."
- You're researching a developer ecosystem and need a quick dashboard rather than raw GH Archive parsing.
- You want to learn TiDB's HTAP patterns from a real workload.

**Avoid when:**
- You want editorial judgement, not raw numbers (use RepoCritics or the wiki you are reading).
- You need real-time event streaming.
- You need data on private repositories, enterprise accounts, or non-GitHub hosts (GitLab, Gitee, sourcehut).
- You're evaluating a specific repo for production use — star trajectories tell you almost nothing about reliability, governance, or maintenance health.

## Alternatives

- [GitHub Insights & GraphQL API](https://docs.github.com/en/graphql) — the canonical source; OSSInsight is downstream.
- [GH Archive](https://www.gharchive.org/) — the raw event firehose. BigQuery has the public mirror.
- [libraries.io](https://libraries.io/) — cross-registry (npm, PyPI, crates, etc.) dependency graph; complementary to GitHub event data.
- [Sourcegraph](https://sourcegraph.com) — code-level search across open source; different angle on the same corpus.
- [ecosyste.ms](https://ecosyste.ms) — open data on packages, repos, and dependencies; community-run, similar spirit.
- RepoCritics — editorial layer on top of GitHub repos; not analytics-first.

## History

| Date | Notes |
|------|-------|
| 2022-05 | OSSInsight launched as a TiDB Cloud showcase[^1]. |
| 2022-09 | Public API stabilized; "Repo Compare" feature. |
| 2023-04 | LLM-assisted SQL exploration ("Data Explorer"). |
| 2023-11 | Self-hosting docs and Docker Compose published. |
| 2024-2025 | Steady additions: collections, GitHub Actions analytics, AI repo summaries. |

## References

[^1]: PingCAP, "Introducing OSSInsight" — 2022-05. https://ossinsight.io/blog/why-we-build-ossinsight
[^2]: OSSInsight architecture overview in the repo README. https://github.com/pingcap/ossinsight#how-it-works
[^3]: GitHub Engineering on star cleanups and abuse detection. https://github.blog/engineering/ (search "spam accounts")
[^4]: PingCAP, "Analyzing 100 billion rows of GitHub events with TiDB". https://www.pingcap.com/blog/
[^5]: Dagster Labs, "GitHub stars are a vanity metric" — 2022. https://dagster.io/blog/fake-stars
