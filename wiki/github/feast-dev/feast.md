# feast-dev/feast

> An open-source feature store: a registry, materialization pipeline, and serving layer that keeps ML features consistent between training and online inference.

[GitHub repo](https://github.com/feast-dev/feast) ·
[Official website](https://feast.dev) ·
[License: Apache-2.0](https://github.com/feast-dev/feast/blob/master/LICENSE)

## Overview

Feast (**Fea**ture **St**ore) solves one narrow, expensive problem: the training/serving skew that appears when the feature values a model saw during training differ from the ones it sees at inference time. It does this by defining features once (in Python), computing point-in-time-correct training sets from an *offline store*, materializing the latest values into a low-latency *online store*, and serving them through a feature server. The registry — a single declarative source of truth — ties the three together.

The project began at Gojek in 2019 in collaboration with Google Cloud, was contributed to the LF AI & Data Foundation in 2020, and for several years was stewarded largely by Tecton (the commercial feature-platform company co-founded by Feast's original author)[^1]. Since ~2023 maintenance has broadened to a wider community, with Red Hat driving the Kubernetes Operator and OpenShift AI integration[^2]. It remains pre-1.0 (0.x) as of 2026.

The defining tension: Feast is deliberately *not* a compute engine. It does not run large-scale feature transformations for you — it orchestrates existing infrastructure (BigQuery, Snowflake, Redshift, Redis, DynamoDB) and pushes work down to it. That makes Feast light to adopt and hard to outgrow at the storage layer, but it means "feature engineering" beyond simple on-demand row transforms is your problem, not Feast's. Teams expecting Tecton's managed streaming compute out of an OSS pip install are the most common source of disappointment.

## Getting Started

```bash
pip install feast
feast init my_feature_repo
cd my_feature_repo/feature_repo
feast apply          # register definitions + provision stores
```

```python
from feast import FeatureStore
import pandas as pd
from datetime import datetime

store = FeatureStore(repo_path=".")

# Point-in-time-correct training set: features joined AS OF each event row
entity_df = pd.DataFrame({
    "driver_id": [1001, 1002],
    "event_timestamp": [datetime(2021, 4, 12, 10, 59), datetime(2021, 4, 12, 8, 12)],
})
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=["driver_hourly_stats:conv_rate", "driver_hourly_stats:avg_daily_trips"],
).to_df()

# After `feast materialize-incremental <now>`, read the same features online:
vector = store.get_online_features(
    features=["driver_hourly_stats:conv_rate", "driver_hourly_stats:avg_daily_trips"],
    entity_rows=[{"driver_id": 1001}],
).to_dict()
```

## Architecture / How It Works

Feast has four moving parts wired together by a *provider* (the glue that maps abstract operations to a concrete cloud — `local`, `gcp`, `aws`):

1. **Registry** — the single source of truth for feature definitions. Historically a serialized protobuf blob on a local disk or object store (S3/GCS); a SQL-backed registry was added later for concurrency and scale. The whole registry is loaded into memory on `FeatureStore()` init.
2. **Offline store** — where historical data lives and where point-in-time joins execute. `get_historical_features` compiles to a query pushed *into* the store (SQL for BigQuery/Snowflake/Redshift, DuckDB/pandas for files). Feast does not move the data through Python; it delegates the join.
3. **Online store** — a key-value store (Redis, DynamoDB, Datastore, SQLite, Postgres, Cassandra, and many contrib backends) holding the latest materialized value per entity, read at single-digit-millisecond latency.
4. **Materialization engine** — moves rows from offline to online. The default engine runs **in-process and single-threaded**; scale-out engines (Snowflake, Bytewax, Spark, AWS Lambda) exist for large loads.

Serving is offered as a Python feature server (FastAPI) plus alpha Go and Java servers for lower-latency paths. Transformations come in two flavors: *on-demand* transforms that run row-wise in Python at request time, and *streaming/push* ingestion. Batch transformation as a first-class feature has been "in progress" for a long time[^3].

The load-bearing idea is that offline and online stores are pluggable behind one API, so a model stays portable as infrastructure changes. The cost of that abstraction is that anything the underlying store can't do, Feast can't do either.

## Production Notes

- **Feast computes almost nothing itself.** On-demand transforms execute per-request in Python and do not scale like a batch engine; heavy feature engineering must happen upstream (dbt, Spark, warehouse SQL) before Feast sees it. Treat Feast as registry + sync + serving, not as a transformation platform.
- **The default materialization engine is single-threaded and in-process.** Materializing large tables from the local engine is slow and memory-bound; production loads need the Snowflake/Spark/Bytewax/Lambda engines, each with its own setup and maturity.
- **The file-based registry is a scaling floor.** It is read whole on init and re-serialized on `apply`; large registries and concurrent writers hit contention. Move to the SQL registry before the team or feature count grows.
- **TTL and event timestamps are silent footguns.** Point-in-time joins depend on correct `event_timestamp` columns and feature-view TTL. A too-short TTL or a timezone-naive timestamp produces nulls rather than errors — you get a "working" pipeline serving empty features.
- **Serving latency is your store's latency plus overhead.** The Python server carries serialization/GIL cost; the Go and Java servers are lower-latency but alpha. Benchmark the actual online store (Redis vs DynamoDB vs Postgres) — Feast does not change its performance envelope.
- **Plugin maturity is uneven.** Core stores (BigQuery, Snowflake, Redshift, Redis, DynamoDB, SQLite) are well-exercised; the long tail of `contrib` sources and stores varies widely in test coverage and should be validated before relying on them.
- **Pre-1.0 churn.** Breaking changes have landed across 0.x minor releases (config schema, provider internals, deprecated Great Expectations validation). Pin the version and read the changelog before upgrading.

## When to Use / When Not

**Use when:**
- You have training/serving skew and need point-in-time-correct joins you'd otherwise hand-roll and get subtly wrong.
- You already run a supported warehouse and KV store and want a thin, portable feature-definition layer over them.
- You want an open, self-hostable feature registry with a Kubernetes Operator path (OpenShift AI / vanilla k8s).

**Avoid when:**
- You expect built-in, scalable feature transformation or streaming compute — Feast orchestrates, it doesn't compute.
- You're fully inside one managed platform (Databricks, Vertex, SageMaker) whose native feature store removes the portability argument.
- You want a turnkey managed SaaS with SLAs and streaming out of the box — that is Tecton's paid product, not OSS Feast.
- You have a handful of models and a simple batch job; a feature store is operational overhead you may not need yet.

## Alternatives

- tecton/tecton — the commercial, managed platform from Feast's original creators; use when you want streaming compute, SLAs, and support included rather than assembled.
- logicalclocks/hopsworks — an integrated feature store with its own online database (RonDB) and platform; use when you want batteries included over a pluggable thin layer.
- featureform/featureform — a "virtual" feature store that layers metadata over existing infra; use when you want an even lighter registry with minimal materialization machinery.
- airbnb/chronon — Spark/Flink-based feature platform with built-in backfill and streaming; use when transformation and backfill compute at scale is the actual bottleneck.
- linkedin/feathr — Spark-native feature engineering DSL; use when your feature logic is heavy and already Spark-centric (note: reduced maintenance activity).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2019-01 | Open-sourced by Gojek with Google Cloud; Spark/Kubernetes-centric architecture[^1]. |
| — | 2020-09 | Contributed to the LF AI & Data Foundation[^4]. |
| 0.10 | 2021-04 | Major re-architecture: local-first, `pip install feast`, providers; dropped mandatory Spark/k8s[^1]. |
| 0.x | 2021–2026 | Pluggable offline/online stores, SQL registry, on-demand transforms, Go/Java servers, k8s Operator[^2]. |

## References

[^1]: Feast — "A State of Feast" / project history and the 0.10 re-architecture. https://feast.dev/blog/a-state-of-feast/
[^2]: Feast Operator for Kubernetes / OpenShift AI. https://github.com/feast-dev/feast/blob/master/infra/feast-operator/README.md
[^3]: Feast README, "Functionality and Roadmap" — batch transformation marked in progress. https://github.com/feast-dev/feast
[^4]: LF AI & Data Foundation — Feast project page. https://lfaidata.foundation/projects/feast/

## Tags

python, machine-learning, mlops, feature-store, ml-infrastructure, data-engineering, online-inference, point-in-time-join, feature-serving, apache-2.0
