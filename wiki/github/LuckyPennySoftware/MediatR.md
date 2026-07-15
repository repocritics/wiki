# LuckyPennySoftware/MediatR

> In-process mediator for .NET — request/response, notifications, and a pipeline — that went commercial in 2025.

[GitHub repo](https://github.com/LuckyPennySoftware/MediatR) ·
[Licensing](https://luckypennysoftware.com/license) ·
License: RPL-1.5 / commercial (was Apache-2.0)

## Overview

MediatR is a small library that implements the mediator pattern in .NET. Instead of a controller or service calling a handler class directly, it constructs a request object and hands it to `IMediator`, which resolves and invokes the matching handler through the DI container. It covers three shapes: request/response (one handler, `IRequest<T>`), notifications (zero-to-many handlers, `INotification`), and streaming (`IStreamRequest<T>`). A pipeline of `IPipelineBehavior` wrappers sits between send and handler for cross-cutting concerns[^1]. It is deliberately narrow — "simple, unambitious," in the repo's own words — and holds no queues, no persistence, and no cross-process transport.

It was created in 2014 by Jimmy Bogard[^2] and for most of its life shipped under Apache-2.0. It is one of the most-installed .NET packages on NuGet and the default backbone of countless "CQRS" and vertical-slice codebases. Its ubiquity outran its ambitions: Bogard himself has repeatedly cautioned that MediatR is a library, not an architecture, and that teams routinely over-apply it[^3].

The defining event of the project's recent history is commercial. In 2025 ownership moved to Lucky Penny Software (Bogard's company), the repository was relocated from `jbogard/MediatR` to `LuckyPennySoftware/MediatR`, and the license changed from Apache-2.0 to the Reciprocal Public License 1.5 (RPL-1.5) with a paid commercial option and a license-key mechanism[^4]. Any evaluation of MediatR today is really an evaluation of that transition — the code is largely the same; the terms are not.

## Getting Started

```bash
dotnet add package MediatR
```

```csharp
// 1. Define a request and its handler
public record Ping(string Message) : IRequest<string>;

public class PingHandler : IRequestHandler<Ping, string>
{
    public Task<string> Handle(Ping request, CancellationToken ct)
        => Task.FromResult($"Pong: {request.Message}");
}

// 2. Register (scans the assembly for handlers)
services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<Program>());

// 3. Send
public class Controller(ISender sender)
{
    public Task<string> Get() => sender.Send(new Ping("hi"));
}
```

For projects that share request/notification types across assemblies (gRPC, API contracts, Blazor), the contracts-only `MediatR.Contracts` package exposes `IRequest`, `INotification`, and `IStreamRequest` without the runtime[^1].

## Architecture / How It Works

`IMediator` is the union of two interfaces you usually depend on separately: `ISender` (`Send`, for request/response and streams) and `IPublisher` (`Publish`, for notifications). `AddMediatR` scans the assemblies you point it at, finds every closed `IRequestHandler<,>`, `INotificationHandler<>`, `IStreamRequestHandler<,>`, and open-generic behavior, and registers them as transient services[^1]. At dispatch time MediatR builds the closed generic handler type from the request's runtime type and asks the container for it — C# generic variance is what lets a single `Send` call route to the right handler.

Requests are one-to-one: exactly one handler per `IRequest<T>`, and a missing or duplicate registration is a runtime error, not a compile error. Notifications are one-to-many: `Publish` resolves every `INotificationHandler<T>` and, by default, awaits them **sequentially** in registration order. You can swap the notification publisher strategy (run handlers in parallel, or continue past exceptions), but the default is the foreach-await that surprises people.

`IPipelineBehavior<TRequest,TResponse>` is the extension point that carries most of MediatR's real-world value: logging, validation (FluentValidation), caching, transactions, and retries are all written as behaviors that wrap the inner handler via a `next()` delegate. Behavior order is registration order, and getting it wrong (validation after the handler, transaction inside the retry) is a common and silent bug. Pre/post processors are a thinner convenience over the same idea.

The cost of all this is indirection. `sender.Send(new Ping(...))` has no static link to `PingHandler`; "go to definition" lands on the interface, not the code that runs. That is the intended decoupling and also the most-cited critique — the call graph moves from the compiler to the DI container.

## Production Notes

**The license change is the headline operational risk, not a footnote.** Versions through v11 are Apache-2.0 and stay that way. Starting with the Lucky Penny era (v12 onward, and firmly by v13, 2025-07-02) the code is RPL-1.5 with a commercial option[^4]. RPL-1.5 is a strong reciprocal (copyleft-style) license that can require you to publish source of derivative work — for many closed-source shops that means buying a commercial license instead. A license key can be supplied via `cfg.LicenseKey`, or the `MEDIATR_LICENSE_KEY` / `LUCKYPENNY_LICENSE_KEY` environment variables (the latter shared with AutoMapper), and unlicensed use logs a warning[^1]. Confirm current pricing and free-tier thresholds against Lucky Penny's site before upgrading; treat the jump past v11 as a licensing decision with legal sign-off, not a routine bump.

**Handler resolution is reflection- and container-driven.** For the overwhelming majority of apps the per-`Send` overhead is irrelevant next to I/O, but it is not free: closed generic type construction plus a container resolve per call, with allocations. Hot in-process fan-out at very high throughput is where alternatives that use source generators (compile-time dispatch) measurably win.

**Assembly scanning is a startup and correctness surface.** Point `AddMediatR` at the wrong assemblies and handlers silently do not register — you get a runtime "no handler for request" only when that path is exercised. There are no compile-time guarantees that every `IRequest<T>` has a handler.

**Notifications carry ordering and failure assumptions.** Default sequential publish means one slow or throwing handler blocks or aborts the rest. If you need isolation, parallelism, or continue-on-error, you must supply a custom publisher — it is not the default.

**Upgrade friction beyond licensing.** v12 reworked DI registration onto `Microsoft.Extensions.DependencyInjection` and removed the older per-container adapter packages, so pre-v12 registration code does not port unchanged. Combined with the license shift, v11→v13+ is the migration that actually costs time.

## When to Use / When Not

**Use when:**
- You want thin controllers/endpoints and cross-cutting concerns (validation, logging, transactions) expressed once as pipeline behaviors.
- You are doing in-process CQRS or vertical-slice architecture and want a uniform send/handle shape.
- Your organization can accept RPL-1.5 or budget a commercial license.

**Avoid when:**
- The app is small enough that a directly-injected handler class is clearer than a request object plus a dispatcher.
- You need real messaging: queues, durable retries, or cross-process delivery — that is a message bus, not a mediator.
- You cannot accept the license terms; v11-and-earlier Apache builds are frozen, and source-generated MIT alternatives now match most of the API.
- You are adopting it as "architecture." Indirection without a decoupling need is net-negative navigability.

## Alternatives

- martinothamar/Mediator — source-generator mediator with a near-identical API, compile-time dispatch, and an MIT license; use when you want the MediatR shape without runtime reflection or licensing fees.
- Cysharp/MessagePipe — high-performance in-process (and inter-process) pub/sub for .NET; use when notification throughput or allocation pressure is the concern.
- MassTransit/MassTransit — actual distributed messaging over transports; use when you need queues, durability, or cross-service delivery, not in-process dispatch.
- rebus-org/Rebus — lightweight service bus for eventing across process boundaries rather than a request pipeline.
- Direct DI (no mediator) — inject the handler interface and call it; use when the indirection buys nothing and you value navigable call graphs.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2014-03-03 | Created by Jimmy Bogard; Apache-2.0[^2]. |
| 9.0 | 2020-10-07 | Mature Apache-2.0 line; widely adopted. |
| 11.0 | 2022-09-30 | Last of the pre-consolidation DI packages. |
| 12.0 | 2023-02-14 | DI reworked onto `Microsoft.Extensions.DependencyInjection`. |
| 13.0 | 2025-07-02 | Lucky Penny era: RPL-1.5 + commercial license, license keys[^4]. |
| 14.0 | 2025-11-20 | Continued under commercial licensing. |
| 14.2 | 2026 | Current line (see NuGet for exact latest)[^5]. |

As of mid-2026 the repository is active — pushed within the last two weeks, ~11.8k stars, ~2.2k forks — with a near-empty issue tracker, consistent with a small, now-commercial project maintained by its owner rather than a large contributor pool.

## References

[^1]: MediatR README and wiki. https://github.com/LuckyPennySoftware/MediatR
[^2]: Repository created 2014-03-03 (GitHub API `created_at`). Original owner Jimmy Bogard (`jbogard/MediatR`, now redirected). https://github.com/jbogard
[^3]: Jimmy Bogard, recurring guidance that MediatR is a library, not an architecture. https://www.jimmybogard.com/
[^4]: License text (LICENSE.md), Reciprocal Public License 1.5 with commercial option via Lucky Penny Software. https://luckypennysoftware.com/license
[^5]: MediatR on NuGet. https://www.nuget.org/packages/MediatR

## Tags

csharp, dotnet, mediator-pattern, cqrs, in-process-messaging, dependency-injection, pipeline-behavior, commercial-license, rpl-1.5, request-response
