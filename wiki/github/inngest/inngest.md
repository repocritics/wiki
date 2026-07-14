# inngest/inngest

> Durable execution engine and dev server — write retryable step functions over events instead of wiring queues, state, and schedulers by hand.

[GitHub repo](https://github.com/inngest/inngest) ·
[Official website](https://www.inngest.com/docs) ·
License: SSPL-1.0 + delayed Apache-2.0 (SPDX reports NOASSERTION)

## Overview

Inngest is a workflow/durable-execution platform written in Go, first published in 2021[^1]. The developer writes ordinary functions in a language SDK (TypeScript, Python, Go, Kotlin/Java); Inngest triggers them from events, cron, or webhooks and guarantees that each `step` inside the function runs to completion with automatic retries, even across process restarts and multi-day sleeps. It occupies the same "durable execution" category as Temporal and Restate, but with a distinguishing design choice: functions are invoked over HTTP rather than run inside long-lived workers, which makes it fit serverless and edge platforms without a persistent connection.

The defining tension is the split between the open-source engine in this repo and the commercial hosted platform. The Dev Server (`inngest-cli`) gives full local parity — event feed, queues, a dashboard, replay — and the same server can be self-hosted. But the server and CLI are licensed under the SSPL with a delayed Apache-2.0 grant (DOSP), not an OSI-approved open-source license[^2]; the SDKs are Apache-2.0. This means you can read, run, and self-host the code, but you cannot offer Inngest itself as a competing managed service. Practically, most teams use the OSS Dev Server locally and the hosted cloud in production, and self-hosting the engine is a supported but younger path.

## Getting Started

Run the Dev Server — no signup, no infra:

```bash
npx inngest-cli@latest dev
# dashboard at http://localhost:8288
```

Define a durable function (TypeScript SDK shown; the engine is language-agnostic):

```typescript
import { Inngest } from "inngest";
const inngest = new Inngest({ id: "my-app" });

export default inngest.createFunction(
  { id: "import-product-images", concurrency: { key: "event.data.userId", limit: 10 } },
  { event: "shop/product.imported" },
  async ({ event, step }) => {
    // each step.run is a durable checkpoint — memoized once it succeeds
    const urls = await step.run("copy-to-s3", () =>
      copyAllImagesToS3(event.data.imageURLs)
    );
    await step.run("resize", () =>
      resizer.bulk({ urls, quality: 0.9, maxWidth: 1024 })
    );
  }
);

// trigger it from anywhere:
await inngest.send({ name: "shop/product.imported", data: { userId, imageURLs } });
```

Your app exposes an HTTP `/api/inngest` endpoint; Inngest calls back into it to drive each step.

## Architecture / How It Works

The server (this repo) is a set of cooperating components, runnable as a single binary[^3]:

- **Event API** — HTTP ingress for events from SDKs, authenticated with Event Keys; publishes to an internal event stream that buffers producers from the runner.
- **Runner** — consumes events, schedules "function runs" (jobs), creates initial run state, resumes functions blocked on `waitForEvent`, and cancels runs matching `cancelOn` expressions.
- **Queue** — a multi-tenant, multi-tier queue built for fairness and flow control: concurrency, throttling, debouncing, rate limiting, prioritization, and batching are all queue-level features, not app code.
- **Executor** — invokes your function over HTTP, persists incremental step state, and retries failed steps.
- **State store / Database** — persists pending run state (triggering events, step outputs, step errors) plus historical records of apps, functions, events, and results.
- **API + Dashboard** — GraphQL and REST for management; a web UI for run history and replay.

The core mechanic is **memoized re-invocation**. Inngest does not keep your function paused in memory. Instead, each time a step needs to run, it re-invokes your function endpoint from the top; steps that already completed return their stored output instead of executing again, and the function fast-forwards to the first unfinished step. `step.sleep`, `step.waitForEvent`, and `step.invoke` end the current invocation entirely and schedule a later one. This is how a function can "run for months" without holding a process — but it means the code between steps re-executes on every invocation. That property is the source of both the model's resilience and its most common footgun.

## Production Notes

**Everything with side effects must live inside a `step`.** Because the function body re-runs on each step, any I/O, random value, timestamp, or mutation written outside `step.run` executes multiple times (once per invocation). Non-deterministic values captured outside a step will differ between replays. This is the same determinism discipline Temporal imposes, surfaced differently.

**Per-step HTTP round-trips cost latency and are size-bounded.** Each step is a separate request/response between the Inngest server and your endpoint. Step output is serialized into run state, so returning large payloads (multi-MB blobs, full DB result sets) bloats state and can hit request limits — return references (S3 keys, IDs), not the data. Functions with hundreds of fine-grained steps pay a round-trip per step.

**Serverless timeouts vs. step count.** On platforms with hard request timeouts (Vercel, Lambda, Cloudflare Workers) a single invocation must finish its current step within that limit. Long individual steps still need a long-enough platform timeout; splitting work into more, smaller steps is the usual mitigation.

**Self-hosting is supported but younger than the cloud.** The Dev Server is excellent for local parity, but running the engine in production means operating its datastore dependencies and accepting that features and hardening land in Inngest Cloud first. Pin the server and SDK versions together and read the self-hosting guide before committing[^4].

**Licensing gate.** The SSPL is a real constraint for anyone whose business is offering managed developer infrastructure — you cannot productize the Inngest server as a service. For internal use and self-hosting it is unproblematic; for "we resell durable execution," read the license[^2].

## When to Use / When Not

**Use when:**
- You want durable, retryable background jobs and multi-step workflows without standing up Kafka/Redis/Temporal yourself.
- You deploy to serverless or edge and can't run long-lived worker processes.
- You need event-driven fan-out, `waitForEvent` coordination, and flow control (concurrency, throttling, debounce) as configuration rather than code.
- You want strong local dev parity for async workflows.

**Avoid when:**
- You need a fully OSI-open, vendor-neutral engine with no service-offering restriction — the SSPL disqualifies it for some.
- Your workflows are extremely fine-grained/high-frequency where per-step HTTP overhead dominates; an in-process queue (BullMQ) or worker-based engine (Temporal) fits better.
- You want language-native, in-worker workflow code with no HTTP callback model.
- You need mature, battle-hardened self-hosting today rather than the hosted platform.

## Alternatives

- temporalio/temporal — the reference durable-execution engine; worker-based, language-native, more operationally heavy. Use instead when you run persistent workers and want maximum maturity.
- restatedev/restate — Rust durable-execution core with a similar serverless-friendly, HTTP-invocation model. Use instead when you want a single self-contained binary and OSI-friendly licensing.
- triggerdotdev/trigger.dev — TypeScript-first background jobs and durable workflows with comparable DX. Use instead when you are TS-only and want an open-source-licensed alternative.
- hatchet-dev/hatchet — Postgres-backed task queue and orchestration. Use instead when you prefer a database-centric queue you fully own.
- taskforcesh/bullmq — Redis-backed job queue, lower level, no durable step-replay model. Use instead when you just need a queue, not workflow orchestration.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2021-06 | Public engine + CLI first pushed[^1]. |
| SDK v3 era | 2023 | Middleware, revised step APIs in the language SDKs. |
| Self-hosting | 2024 | Self-hosting the server documented as a supported path[^4]. |
| Active | 2026-07 | Ongoing; ~5.6k stars, frequent commits[^1]. |

## References

[^1]: inngest/inngest repository — created 2021-06-07, ~5,605 stars / 326 forks, last pushed 2026-07 (GitHub API, fetched 2026-07-15). https://github.com/inngest/inngest
[^2]: Inngest license — Server Side Public License with delayed open-source publication under Apache-2.0; SDKs Apache-2.0. https://github.com/inngest/inngest/blob/main/LICENSE.md
[^3]: Inngest README, "Project Architecture" — Event API, Runner, Queue, Executor, State store, Dashboard. https://github.com/inngest/inngest
[^4]: Inngest docs, "Self-hosting". https://www.inngest.com/docs/self-hosting

## Tags

go, durable-execution, workflow-engine, event-driven, background-jobs, step-functions, serverless, queues, orchestration, sspl
