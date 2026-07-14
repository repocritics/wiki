# windmill-labs/windmill

> Open-source developer platform that turns scripts (Python, TypeScript, Go, Bash, SQL…) into APIs, scheduled jobs, workflows, and auto-generated UIs — with Postgres as the job queue.

[GitHub repo](https://github.com/windmill-labs/windmill) ·
[Official website](https://windmill.dev) ·
[License: AGPL-3.0](https://github.com/windmill-labs/windmill/blob/main/LICENSE-AGPL) (open-core; see licensing note)

## Overview

Windmill is a self-hostable platform for internal engineering: you write a small script with a typed `main` function, and Windmill parses its signature to generate an input form, a webhook, a schedule surface, and a permissioned execution environment automatically[^1]. Scripts compose into flows (DAGs) and low-code apps. It positions itself as an open-source alternative to Retool (UI builder), Pipedream/n8n (automation), and a simplified Temporal (durable workflows) — three product categories fused into one runtime.

The defining architectural decision is that **Postgres is the job queue**. Stateless Rust API servers and workers pull jobs from a Postgres table rather than from Redis, RabbitMQ, or a dedicated broker[^2]. This collapses the operational surface to a single stateful dependency, which is Windmill's biggest selling point for small teams and its most-debated scaling characteristic at high job throughput. The company also self-reports being the fastest self-hostable workflow engine (claiming ~13x Airflow on their own benchmark)[^3] — a vendor benchmark, so read it as directional rather than independent.

The project is open-core, not purely open-source. The AGPLv3 binary compiled without the `enterprise` feature flag is fully open; the "Community Edition" Docker images and GitHub binary releases bundle additional proprietary Enterprise code that is free to use internally but restricted for resale, managed-service hosting, or white-labeling[^4]. This licensing nuance is the single most important thing to understand before building a product on top of it.

## Getting Started

Self-host with the published Docker Compose stack (three files):

```bash
curl https://raw.githubusercontent.com/windmill-labs/windmill/main/docker-compose.yml -o docker-compose.yml
curl https://raw.githubusercontent.com/windmill-labs/windmill/main/Caddyfile -o Caddyfile
curl https://raw.githubusercontent.com/windmill-labs/windmill/main/.env -o .env
docker compose up -d
# http://localhost — default login: admin@windmill.dev / changeme (change it)
```

A script is an exported `main`; its typed parameters become the auto-generated UI and the webhook payload schema:

```typescript
import * as wmill from "windmill-client";

export async function main(
  repo: string,
  limit = 10,            // default arg → inferred type + form default
) {
  // variables/secrets are permissioned by path, not env-injected
  const token = await wmill.getVariable("f/github/token");
  const res = await fetch(`https://api.github.com/repos/${repo}/issues?per_page=${limit}`, {
    headers: { authorization: `Bearer ${token}` },
  });
  return await res.json(); // return value is serialized as JSON
}
```

## Architecture / How It Works

- **Backend (Rust).** Stateless components in a single binary whose role is set by `MODE`: `server` (API + web), `worker` (job executor), `agent`, or `standalone` (both). Servers and workers coordinate only through Postgres — there is no separate control plane. You scale by adding worker processes (rule of thumb: 1 worker per vCPU, 1–2 GB RAM each) and grouping them into worker groups with independent config[^2].
- **Queue = Postgres.** Jobs are rows. Workers poll with `SLEEP_QUEUE` (default 50 ms) backoff. Zombie jobs (workers that stop pinging) are detected by a periodic server sweep and either restarted in place or failed (`RESTART_ZOMBIE_JOBS`, `ZOMBIE_JOB_TIMEOUT`). Post-startup, per-job overhead is ~50 ms of queue latency plus runtime start[^1].
- **Language runtimes.** TypeScript/JavaScript runs on Bun (default) or Deno; Python uses `uv` for dependency resolution; Go, Bash, PowerShell, PHP, Rust, C#, Java, SQL, GraphQL, and Ansible are also supported. Dependencies are declared inline (imports) and cached per worker.
- **Sandboxing.** Jobs run under google/nsjail for filesystem and resource isolation, plus PID-namespace isolation (on by default) so a job cannot read the worker process memory[^5]. nsjail requires elevated container privileges, which matters on locked-down Kubernetes.
- **Frontend.** The web IDE, flow editor, and app builder are a Svelte 5 SPA. Apps can be embedded/iframed; deeper white-label re-exposure crosses into the commercial-license territory noted above.
- **Secrets.** One encryption key per workspace protects credentials stored in Windmill's own K/V store in Postgres; encrypting the database at rest is recommended, not automatic.

## Production Notes

- **Postgres is the whole system.** Its availability, connection limits, and IOPS are your ceiling. High-frequency polling from many workers generates constant query load; very high job rates or thousands of workers push you toward a larger instance, `PgBouncer`, or worker-group tuning. There is no built-in horizontal Postgres story — this is the trade for having no broker.
- **Release cadence is very fast.** Windmill ships continuously under a single incrementing `v1.x` line (v1.757.0 in mid-2026)[^6]. There are no semantic major versions; "upgrade" means moving forward many minor releases at once. Pin image tags, read changelogs, and test — do not blindly track `latest` in production.
- **Enterprise feature flags.** Metrics (`METRICS_ADDR`/Prometheus), some scaling and audit features, and others are EE-gated and require a `LICENSE_KEY`. Confirm a capability is in the AGPL build before designing around it; the docs do not always foreground the CE/EE boundary.
- **nsjail + privileges.** The default sandbox needs kernel features that some managed Kubernetes and restrictive PaaS environments disallow. Verify isolation works in your target before relying on multi-tenant safety.
- **State and idempotency.** `getState`/`setState` give scripts lightweight persistence, but Windmill is not a full durable-execution engine like Temporal — retries and flow steps are at-least-once, so make side-effecting steps idempotent yourself.
- **Self-hosted vs. cloud.** app.windmill.dev is the managed offering; self-hosting is a first-class supported path (Docker Compose, Helm charts, most clouds), which is unusual and welcome for this category.

## When to Use / When Not

**Use when:**
- You want one tool for internal APIs, cron jobs, workflows, and admin UIs without stitching four systems together.
- You prefer code-first scripts (real Python/TS with dependencies) over node-dragging automation.
- Operational minimalism matters: Postgres + Windmill binaries, nothing else to run.
- You need self-hosting with permissioned secrets and sandboxed execution.

**Avoid when:**
- You need guaranteed durable execution with strong exactly-once/replay semantics — Temporal is purpose-built for that.
- Your workload is heavy data-engineering DAGs with rich data-aware scheduling — Airflow/Dagster fit better.
- You want a pure MIT/Apache OSS with no proprietary Enterprise layer or resale restrictions.
- You expect extreme job throughput where a single Postgres queue becomes the bottleneck.

## Alternatives

- n8n-io/n8n — node-based visual automation with a large integration catalog; use it when non-developers build the workflows and code-first isn't required (note: fair-code, not OSI-approved).
- temporalio/temporal — durable execution engine with replay guarantees; use it when correctness of long-running stateful workflows is the priority over an all-in-one UI.
- apache/airflow — Python DAG scheduler for data pipelines; use it for batch data-engineering with mature ecosystem operators.
- PrefectHQ/prefect — Python-native dataflow orchestration; use it when you want orchestration as a library inside existing Python code.
- appsmithorg/appsmith — open-source internal-tool/UI builder; use it when the deliverable is mostly admin UIs over databases rather than backend jobs.

## History

Windmill uses a continuous single-line versioning scheme (`v1.<n>`) with frequent releases rather than semantic majors, so milestones below are directional.

| Version | Date | Notes |
|---------|------|-------|
| — | 2022-05 | Repository open-sourced; Rust backend + Postgres queue foundation[^7]. |
| v1.x | 2022–2024 | Flows, apps/low-code builder, Hub, CLI and VS Code sync added incrementally. |
| v1.x | ~2024 | Bun becomes the default TypeScript runtime alongside Deno; `uv` for Python. |
| v1.x | ~2024–2025 | Frontend on Svelte 5; expanded language runtimes (PHP, Rust, C#, Java, Ansible). |
| v1.757.0 | 2026-07-14 | Latest tagged release at time of writing[^6]. |

## References

[^1]: Windmill README — main concepts, auto-generated UIs, performance (~50 ms per-job overhead). https://github.com/windmill-labs/windmill
[^2]: Windmill docs — architecture: stateless Rust servers/workers pulling jobs from a Postgres queue. https://www.windmill.dev/docs/advanced/self_host
[^3]: Windmill benchmarks vs. Airflow/Prefect/Temporal (vendor-run). https://www.windmill.dev/docs/misc/benchmarks/competitors
[^4]: Windmill licensing — AGPLv3 open-source binary vs. Community/Enterprise Edition proprietary terms. https://github.com/windmill-labs/windmill/blob/main/LICENSE-AGPL
[^5]: Windmill security & isolation docs — nsjail and PID-namespace isolation. https://www.windmill.dev/docs/advanced/security_isolation
[^6]: Windmill GitHub releases. https://github.com/windmill-labs/windmill/releases
[^7]: Repository created 2022-05-05 (GitHub API metadata).

## Tags

rust, workflow-engine, low-code, job-scheduler, self-hostable, postgresql, internal-tools, developer-platform, open-core, svelte, python, typescript
