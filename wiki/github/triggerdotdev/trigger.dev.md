# triggerdotdev/trigger.dev

> Open-source durable background-task platform for TypeScript — long-running jobs with retries, queues, schedules, and checkpoint-based waits, deployed from your own codebase.

[GitHub repo](https://github.com/triggerdotdev/trigger.dev) ·
[Official website](https://trigger.dev) ·
[License: Apache-2.0](https://github.com/triggerdotdev/trigger.dev/blob/main/LICENSE)

## Overview

Trigger.dev is a platform for writing background jobs — now marketed primarily as "AI agents and workflows" — as ordinary TypeScript functions that live in your application repo and run on managed (or self-hosted) infrastructure[^1]. The core promise is durability: a task can run for hours, retry on failure, pause on a schedule or a human approval, and survive process restarts, without you operating a queue, a worker pool, or a state machine yourself.

The defining technical bet is the **checkpoint-resume system**. When a task hits a long wait (`wait.for()`, a scheduled delay, a wait-for-token), Trigger.dev snapshots the running process state, frees the compute, and restores the process when the wait resolves[^2]. This is what lets a task "sleep for 3 days" without holding a paid worker open the whole time — the mechanism that distinguishes it from a plain job runner like BullMQ. It is also the source of most of the platform's operational complexity, because checkpointing depends on the execution environment supporting it.

The project has been rewritten twice. The current line, **v3**, is architecturally unrelated to the v2 "jobs + integrations" model that preceded it. Understanding which version a tutorial, StackOverflow answer, or old blog post refers to is the single biggest source of confusion for newcomers, because v2 and v3 share a brand but almost no API.

## Getting Started

```bash
# In an existing TS project
npx trigger.dev@latest init
```

```ts
// src/trigger/example.ts
import { task, wait } from "@trigger.dev/sdk";

export const sendReminder = task({
  id: "send-reminder",
  // automatic retries on uncaught errors
  retry: { maxAttempts: 3 },
  run: async (payload: { userId: string }) => {
    await wait.for({ hours: 24 }); // checkpointed — no compute held
    return { notified: payload.userId };
  },
});
```

```bash
npx trigger.dev@latest dev      # run tasks locally against the platform
npx trigger.dev@latest deploy   # build + ship to cloud or self-hosted instance
```

Tasks must be exported from files under the configured `trigger` directory; the CLI discovers them by static analysis at build time, so dynamically-constructed task definitions will not register.

## Architecture / How It Works

Trigger.dev is a monorepo (pnpm) whose shipped product has several parts:

1. **SDK** (`@trigger.dev/sdk`) — the `task()`, `schedules.task()`, `wait`, `metadata`, and trigger/`batchTrigger` APIs you write against. Payloads and outputs can be validated with `schemaTask` (Zod and other schema libraries).
2. **CLI** — builds your tasks with esbuild via a **build-extension** system, so you can inject system packages (Python, FFmpeg, headless browsers) into the task container image, then deploys them.
3. **Webapp / dashboard** — a Remix application backed by PostgreSQL (application state) and Redis (queueing), providing the run list, trace view, alerting, and environment management. Newer versions add further services for run history and observability.
4. **Worker / supervisor layer** — pulls runs off queues, boots the task in an isolated environment, and drives the checkpoint-resume lifecycle.

Each run produces a full **OpenTelemetry-style trace**: every task, subtask (`triggerAndWait`), and instrumented call appears as a span, which is the observability story that a raw queue does not give you. **Atomic versioning** means a deploy creates a new immutable version; runs already in flight finish on the version they started on, so a deploy never mutates running work.

The conceptual model is closest to durable-execution engines (Temporal, Inngest, Restate): your `run` function is expected to be resumable, and the platform re-enters it around wait points. Unlike Temporal's explicit workflow/activity split and determinism constraints, Trigger.dev leans on process checkpointing rather than deterministic replay, which makes ordinary imperative code "just work" but ties durability to infrastructure capability.

## Production Notes

**Self-hosting is real but not at parity with cloud.** There are official Docker Compose and Kubernetes (Helm) guides[^3], but the managed cloud gets features first and the checkpoint/resume path has historically been the hardest part to run yourself — long waits may keep a worker resident when checkpointing is unavailable in your environment, changing the cost/behavior profile versus the hosted product. Budget real time for the self-hosting setup; it is not a single-container deploy.

**The v2 → v3 migration is a rewrite, not an upgrade.** There is no automatic codemod from the v2 `client.defineJob` + integrations model to v3 `task()`; the integration abstraction was removed entirely in favor of calling SDKs directly inside tasks. Teams on v2 effectively rewrite their jobs. v2 is legacy — new work should start on v3/`@trigger.dev/sdk`.

**Task discovery is build-time and static.** Tasks are found by exported declarations, not at runtime. Conditionally-defined or generated tasks will silently fail to register. Keep one `task` per export, top-level.

**Concurrency and queues are first-class but plan-bound on cloud.** Per-environment concurrency limits, named queues, and idempotency keys exist in the SDK; on the managed service the concurrency ceiling is tied to your billing tier, which surfaces as queued (not failed) runs when exceeded.

**Cold boots.** A run starts by booting an isolated environment for your deployed code. For infrequently-triggered tasks expect boot latency before your `run` body executes; it is not a warm in-process call like a local function queue.

**Observability is a genuine strength, but retention and log volume matter.** The trace view is detailed; high-frequency tasks emitting large logs/metadata will accumulate quickly, so treat `metadata` and log verbosity as things with a cost.

## When to Use / When Not

**Use when:**
- You need long-running, retryable background work in TypeScript without operating your own queue + worker + state store.
- You want human-in-the-loop pauses, cron schedules of arbitrary length, or fan-out/`batchTrigger` with built-in tracing.
- You want tasks versioned and co-located with your app code, deployed via CLI.
- You want durable "wait for days" semantics without paying for idle compute (on cloud).

**Avoid when:**
- Your jobs are short, high-throughput, and latency-sensitive — a plain Redis queue (BullMQ) is lighter and cheaper.
- You need a polyglot workflow engine across many languages — Trigger.dev is TypeScript-first.
- You require fully air-gapped, feature-complete self-hosting today — expect cloud-first feature timing and non-trivial ops.
- You want zero vendor concepts: the durable-execution model and platform lifecycle are an abstraction you buy into.

## Alternatives

- inngest/inngest — closest competitor; durable step functions in TS/Python, serverless-friendly, step-based determinism instead of process checkpointing. Prefer when you want a step/event model and easy serverless hosting.
- temporalio/temporal — heavier, polyglot, deterministic-replay durable execution engine. Use when you need multi-language workflows and are willing to run/operate more.
- restatedev/restate — newer durable-execution runtime (single binary, low-latency). Use when you want durable RPC/handlers with minimal infra.
- hatchet-dev/hatchet — Postgres-backed task queue and orchestration. Use when you want a self-hostable queue with DAG orchestration and no external durable-execution model.
- taskforcesh/bullmq — low-level Redis job queue. Use when you only need queues/retries and will build observability and scheduling yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2022-11-30 | Initial public repository[^1]. |
| v2 | 2023 | Jobs defined with `client.defineJob`, event triggers, delays, and first-party "integrations" (Slack, GitHub, etc.). Now legacy. |
| v3 developer preview | late 2023 | Full rewrite: `task()` model, checkpoint-resume durability, CLI build/deploy, integrations abstraction dropped[^2]. |
| v3 GA | 2024 | General availability of the v3 platform and `@trigger.dev/sdk`[^4]. |
| present | 2026-07 | ~15.6k stars; active; AI-agent/workflow positioning; Docker + Kubernetes self-hosting[^3]. |

## References

[^1]: Trigger.dev README and repository metadata. https://github.com/triggerdotdev/trigger.dev
[^2]: Trigger.dev docs, "How it works" — architecture and the checkpoint-resume system. https://trigger.dev/docs/how-it-works
[^3]: Trigger.dev docs, "Self-hosting overview" (Docker Compose and Kubernetes/Helm). https://trigger.dev/docs/self-hosting/overview
[^4]: Trigger.dev changelog. https://trigger.dev/changelog

## Tags

typescript, background-jobs, durable-execution, task-queue, workflow-automation, job-scheduler, ai-agents, self-hostable, orchestration, observability, apache-2.0
