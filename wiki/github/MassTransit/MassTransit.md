# MassTransit/MassTransit

> Message-based distributed application framework for .NET — an opinionated abstraction over RabbitMQ, Azure Service Bus, SQS, and a SQL transport.

[GitHub repo](https://github.com/MassTransit/MassTransit) ·
[Official website](https://masstransit.io) ·
[License: Apache-2.0](https://github.com/MassTransit/MassTransit/blob/develop/LICENSE)

## Overview

MassTransit is a service-bus framework for .NET that sits between your application code and a message broker. Rather than calling a broker client directly, you define message contracts (usually C# interfaces), write consumers, and let MassTransit own serialization, routing topology, retries, redelivery, dead-lettering, and saga state. It has been developed since 2010, primarily by Chris Patterson, making it one of the oldest continuously maintained messaging libraries in the .NET ecosystem[^1].

The framework's defining trait is that it is opinionated about broker topology. On RabbitMQ it creates its own exchanges and queues named after message types and wires publish/subscribe fan-out automatically; on Azure Service Bus and SQS it maps the same conventions onto topics and subscriptions. This buys you a uniform programming model across very different transports — the same consumer code runs on RabbitMQ, Azure Service Bus, Amazon SQS, ActiveMQ, or the built-in SQL transport. The cost is that the broker becomes a MassTransit-shaped broker: interoperating with non-MassTransit producers or consumers means understanding its message envelope and topology conventions.

The most consequential recent development is licensing. In 2025 the maintainer announced that MassTransit v9 would move to a commercial license, with the Apache-2.0 v8 line remaining free and supported for a defined transition period[^2]. Teams starting new work in 2026 should treat "which version, under which license" as a first-class architectural decision, not an afterthought.

## Getting Started

```bash
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
```

```csharp
// Program.cs — .NET host with MassTransit registered via Microsoft DI
using MassTransit;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderSubmittedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        cfg.ConfigureEndpoints(context); // auto-creates queues/exchanges
    });
});

var app = builder.Build();
app.Run();

public record OrderSubmitted(Guid OrderId, string CustomerId);

public class OrderSubmittedConsumer : IConsumer<OrderSubmitted>
{
    public Task Consume(ConsumeContext<OrderSubmitted> context)
    {
        // context.Message.OrderId — the deserialized contract
        return context.Publish(new OrderConfirmed(context.Message.OrderId));
    }
}

public record OrderConfirmed(Guid OrderId);
```

The bus starts and stops as an `IHostedService`; `ConfigureEndpoints` reads registered consumers and provisions broker resources on startup.

## Architecture / How It Works

The core object is the **bus**, a send/publish endpoint plus a set of **receive endpoints** (one per queue). Messages flow through a middleware pipeline — retry, redelivery, outbox, concurrency limiting, and instrumentation are all pipe filters. The pipeline library (formerly the separate GreenPipes project) was folded into the core in v8[^3].

Key building blocks:

- **Message contracts** — plain classes/records or interfaces. Publishing an interface makes MassTransit generate a backing type via a dynamic proxy. Routing is keyed off the message's .NET type name (namespace included), which is why moving a type to a different namespace silently breaks routing between old and new deployments.
- **Consumers** — `IConsumer<T>`; one message type in, side effects and further publishes out. Registered in DI and bound to endpoints.
- **Sagas / state machines** — long-running workflows. The state-machine DSL (formerly the separate Automatonymous project) was merged into MassTransit in v8[^3]. Saga state persists via EF Core, MongoDB, Redis, Marten, Dapper, DynamoDB, and others.
- **Request/response** — request clients give an `async`/`await` RPC feel over asynchronous messaging, with correlation and timeouts handled for you.
- **Transactional outbox** — bus outbox and EF Core outbox implementations mitigate the dual-write problem (committing a DB change and publishing a message atomically)[^4].
- **Riders** — Kafka and Azure Event Hubs attach to the bus as "riders" rather than full transports; they consume streams alongside broker messaging but don't get the full request/response and saga topology treatment.

Serialization defaults to a JSON **envelope**: the wire message wraps your payload with metadata (message types, correlation IDs, host info, headers). This envelope is what enables polymorphic delivery and interface contracts, and it is also the main interop obstacle — external systems must either speak the envelope or you must switch to a raw serializer.

Since v8, MassTransit requires `Microsoft.Extensions.DependencyInjection`. Earlier support for third-party containers and the standalone `Bus.Factory` self-hosting style was removed, standardizing configuration on the .NET generic host[^3].

## Production Notes

**Topology is a coupling contract.** MassTransit provisions exchanges/queues on startup using its naming conventions. This is convenient in greenfield systems and a liability in brownfield ones: publishing to or consuming from queues owned by non-MassTransit apps requires manually overriding entity names, exchange types, and disabling conventional topology. Budget real time for interop.

**Message type names are load-bearing.** Because routing derives from the full type name, refactoring a namespace, renaming a contract, or moving contracts between assemblies can partition a rolling deployment — old and new instances stop seeing each other's messages. Contract types are typically pushed into a shared, rarely-changed assembly for this reason.

**Poison messages and observability.** Failed messages land in `_error` queues; messages with no consumer land in `_skipped`. These fill silently unless monitored. Retry and redelivery policies are configured per endpoint; getting the retry/redelivery/circuit interaction right under real broker outages takes tuning, not defaults.

**The v7 → v8 migration was large.** Automatonymous and GreenPipes merged into the core, the DI container story was simplified to MS DI only, and configuration APIs changed. Upgrading legacy codebases is a project, not a package bump.

**SQL transport is a real option now.** The PostgreSQL/SQL Server transport lets teams run message-based workloads without standing up a broker, using the database they already operate. It trades broker throughput characteristics for operational simplicity; evaluate it against RabbitMQ under your actual load rather than assuming parity[^5].

**Licensing is now a planning item.** With v9 moving to a commercial model and v8 remaining Apache-2.0, an upgrade decision is also a procurement decision[^2]. Pin versions deliberately and read the current license terms before adopting v9 in production.

**Issue tracker is deliberately quiet.** The repo often shows a near-zero open-issue count — this reflects policy, not inactivity: the maintainer closes or converts non-bug reports to GitHub Discussions and asks that Issues be reserved for confirmed bugs. Support happens in Discussions and Discord.

## When to Use / When Not

**Use when:**
- You're building message-based .NET services and want sagas, retries, outbox, and request/response as first-class, transport-agnostic primitives.
- You may switch or run multiple brokers (RabbitMQ in dev, Azure Service Bus in prod) and want one programming model across them.
- You want mature state-machine workflows and saga persistence without hand-rolling correlation and concurrency.

**Avoid when:**
- You need in-process mediation only (no broker) — a mediator library is lighter.
- You must interoperate heavily with non-MassTransit producers/consumers and can't own the topology.
- You want a permanently free, minimal-dependency bus and are unwilling to adopt the v9 commercial license or stay pinned to v8 long-term.
- You want thin, explicit control over a single broker's client — the abstraction will fight you.

## Alternatives

- Particular/NServiceBus — the long-standing commercial .NET service bus; closest feature peer, with a paid support model. Use when you want vendor-backed support and tooling.
- rebus-org/Rebus — lightweight, free (MIT) .NET service bus with fewer conventions. Use when you want a simpler, permissively licensed bus post-v9.
- JasperFx/Wolverine — newer messaging + in-process mediator with tight Marten integration and code-first handlers. Use when you want lower ceremony and a Postgres-based outbox.
- dotnetcore/CAP — event bus centered on the outbox pattern with broker integrations. Use when reliable event publishing with local-transaction guarantees is the priority.
- Raw clients (RabbitMQ.Client, Azure.Messaging.ServiceBus, Confluent.Kafka) — no framework. Use when you want full control of one transport and minimal abstraction.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010-07 | Project started; early transports included MSMQ and RabbitMQ[^1]. |
| 8.0 | 2022 | Major consolidation: Automatonymous and GreenPipes merged into core; MS DI required; modern .NET targets[^3]. |
| 8.1 | 2023 | SQL transport (PostgreSQL / SQL Server) introduced — broker-less messaging[^5]. |
| 9.0 | 2025–2026 | Announced move to a commercial license; v8 remains Apache-2.0 during transition[^2]. |

## References

[^1]: MassTransit repository, created 2010-07-16; ~7.8k stars, ~2.0k forks, Apache-2.0 as of 2026-07 (GitHub API). https://github.com/MassTransit/MassTransit
[^2]: MassTransit commercial licensing announcement (v9), 2025 — v8 remains Apache-2.0 during the transition. https://masstransit.io/
[^3]: MassTransit v8 documentation — Automatonymous/GreenPipes consolidation and MS DI requirement. https://masstransit.io/documentation/concepts
[^4]: MassTransit transactional outbox documentation. https://masstransit.io/documentation/patterns/transactional-outbox
[^5]: MassTransit SQL transport (PostgreSQL / SQL Server) documentation. https://masstransit.io/documentation/transports/sql

## Tags

csharp, dotnet, messaging, service-bus, rabbitmq, azure-service-bus, amazon-sqs, saga, distributed-systems, event-driven, message-queue
