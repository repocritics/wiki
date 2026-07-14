# langfuse/langfuse

> Open-source LLM engineering platform — tracing/observability, evals, prompt management, and a playground, self-hostable on your own infrastructure.

[GitHub repo](https://github.com/langfuse/langfuse) ·
[Official website](https://langfuse.com) ·
[License: MIT (except `ee/`)](https://github.com/langfuse/langfuse/blob/main/LICENSE)

## Overview

Langfuse is an application-layer observability and evaluation platform for LLM
apps: you instrument your code, ship traces of every model call (plus retrieval,
embedding, and agent steps around them), and get a UI to inspect sessions,
score outputs, manage prompt versions, and run offline evals against datasets[^1].
It began in the YC W23 batch and, per the project's own README, became part of
ClickHouse in January 2026[^2] — which is more than a footnote, because
ClickHouse is also the database Langfuse stores its trace analytics in.

The defining tension is **observability tool vs. self-hosted data platform**.
Langfuse markets "self-host in minutes," and for a Docker Compose demo that is
true. But the thing you are hosting is a distributed data system — a Next.js web
app, a background worker, Postgres, ClickHouse, Redis, and S3-compatible blob
storage — and running that reliably at trace volume is a real operations
commitment, not a single container. Teams routinely start on Langfuse Cloud and
only later confront the self-hosting footprint. Langfuse is deliberately
integration-broad rather than framework-specific: typed Python/JS SDKs, an OpenAI
drop-in wrapper, LangChain/LlamaIndex callback handlers, and OpenTelemetry
ingestion let it sit downstream of almost any stack rather than adopting one[^1].

## Getting Started

Instrument an app against Langfuse Cloud or a self-hosted instance:

```bash
pip install langfuse openai
```

```bash
# .env
LANGFUSE_SECRET_KEY="sk-lf-..."
LANGFUSE_PUBLIC_KEY="pk-lf-..."
LANGFUSE_BASE_URL="https://cloud.langfuse.com"   # or your self-hosted URL
```

```python
from langfuse import observe
from langfuse.openai import openai   # drop-in OpenAI wrapper, auto-captures params

@observe()                            # decorator creates a trace/span
def story():
    return openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": "What is Langfuse?"}],
    ).choices[0].message.content

story()
```

Self-host a full stack locally:

```bash
git clone --depth=1 https://github.com/langfuse/langfuse.git
cd langfuse
docker compose up      # web + worker + postgres + clickhouse + redis + minio
```

## Architecture / How It Works

Langfuse is a TypeScript monorepo (Turborepo) with a Next.js **web** container
and a separate **worker** container, backed by four data stores. The v3 rewrite
(late 2024) is the architecture that matters today; v2 was a simpler
Postgres-only design and is a different operational animal[^3].

- **Postgres** — transactional/config data: users, projects, API keys, prompts,
  dataset definitions, and other OLTP state.
- **ClickHouse** — the analytics store for traces, observations, and scores. This
  is what makes high-cardinality trace queries and dashboards fast, and why the
  v3 stack is heavier than v2[^3].
- **Redis** — queue and cache. Ingestion is asynchronous: SDK events are buffered
  and processed off the hot path.
- **S3 / blob storage** — raw event payloads (large prompts/completions) are
  written to object storage rather than a database row.

**Ingestion is async by design.** The SDK batches events and returns immediately;
the worker consumes the queue, writes blobs to S3, and materializes rows into
ClickHouse. This keeps client-side overhead low, but traces appear with a lag and
a backed-up worker or Redis outage surfaces as missing/delayed data rather than
as errors in your app. The SDKs increasingly speak **OpenTelemetry**, so much of
the integration surface is "point your existing tracing at Langfuse" rather than
a proprietary protocol. Prompt management is a versioned store with aggressive
server- and client-side caching so fetching a prompt's production label adds no
request latency.

## Production Notes

**Self-hosting is a five-service system, not an app.** The "minutes" claim covers
`docker compose up`; a durable deployment means running Postgres, ClickHouse,
Redis, and S3 as managed/HA services. Langfuse's own guidance names **Kubernetes
via the Helm chart** as the preferred production path, with Terraform templates
for AWS/Azure/GCP[^4]. ClickHouse is the component most teams have never operated
before and is the usual source of self-hosting pain.

**v2 → v3 is a migration, not an upgrade.** Moving from the Postgres-only v2 to
the ClickHouse-based v3 requires standing up the new stores and running a data
migration; plan for it explicitly rather than bumping a tag[^3].

**Cost and retention scale with trace volume.** Every LLM call can produce
multiple observations plus payloads; at production traffic, ClickHouse storage
and S3 blob growth are the budget drivers. Sampling, retention policies, and
truncating large payloads are the standard levers.

**The `ee/` license carve-out.** The repository is MIT **except** the `ee/`
folders, which hold commercial enterprise features[^5] — the reason GitHub
reports the license as `NOASSERTION`. If you self-host, audit which features live
under `ee/` (SSO enforcement and some enterprise RBAC controls have historically
been gated) before assuming everything in the repo is MIT. Langfuse Cloud (EU/US
regions, free tier) reaches near parity with self-hosting, but this gate plus
Cloud-only conveniences mean you should read the docs before assuming it.

## When to Use / When Not

**Use when:**
- You need LLM tracing + evals + prompt management in one tool, with a self-host
  option for data residency.
- You are multi-framework (LangChain, LlamaIndex, raw SDK, OTel) and want one sink.
- You want prompt versioning with production labels and low-latency cached fetches.
- You need dataset-based offline evals and LLM-as-a-judge scoring next to traces.

**Avoid when:**
- You want a zero-ops add-on and won't run (or pay for) ClickHouse + Redis + S3.
- Your needs are pure infra tracing — a general OTel + Grafana/Datadog stack fits.
- You need enterprise SSO/RBAC self-hosted for free — check the `ee/` gating first.
- You are at trivial volume and a few `print`/log lines would do.

## Alternatives

- traceloop/openllmetry — OpenTelemetry-native LLM instrumentation; use when you
  want to stay vendor-neutral and feed existing OTel backends rather than a UI.
- Arize-ai/phoenix — open-source LLM tracing/eval with strong retrieval/RAG
  debugging; use when embeddings and RAG analysis are your primary concern.
- comet-ml/opik — Comet's open-source LLM eval/observability platform; use when
  you want a similar feature set with Comet's experiment-tracking lineage.
- Helicone/helicone — proxy-based observability; use when you prefer a gateway/
  proxy that captures calls with near-zero code changes over SDK instrumentation.
- LangSmith (LangChain, closed source) — use when you are all-in on LangChain and
  want first-party integration and don't need self-hosting.

## History

| Version | Date | Notes |
|---------|------|-------|
| Launch | 2023-05 | Initial release; YC W23 batch[^2]. |
| v2 | 2024 | Postgres-only architecture; broad SDK/framework integrations. |
| v3 | 2024-12 | ClickHouse + Redis + S3 async ingestion; web/worker split[^3]. |
| — | 2026-01 | Langfuse becomes part of ClickHouse[^2]. |

## References

[^1]: Langfuse documentation — tracing, prompt management, evaluations, integrations. https://langfuse.com/docs
[^2]: langfuse/langfuse README — YC W23; "since January 2026 we're part of ClickHouse." https://github.com/langfuse/langfuse
[^3]: Langfuse self-hosting — architecture and the v3 (ClickHouse-based) infrastructure. https://langfuse.com/self-hosting
[^4]: Langfuse self-hosting — Kubernetes (Helm) as the preferred production deployment; AWS/Azure/GCP Terraform templates. https://langfuse.com/self-hosting/kubernetes-helm
[^5]: langfuse/langfuse LICENSE — MIT except the `ee/` folders. https://github.com/langfuse/langfuse/blob/main/LICENSE

## Tags

typescript, llm-observability, llmops, tracing, evaluation, prompt-management, self-hosted, clickhouse, opentelemetry, ai-engineering, monitoring, open-source
