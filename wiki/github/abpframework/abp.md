# abpframework/abp

> An opinionated ASP.NET Core application framework — DDD layering, a module system, and multi-tenancy baked in, with a commercial layer sitting on top of the open-source core.

[GitHub repo](https://github.com/abpframework/abp) ·
[Official website](https://abp.io) ·
[License: LGPL-3.0](https://github.com/abpframework/abp/blob/dev/LICENSE)

## Overview

ABP is a full application framework for ASP.NET Core, built by Volosoft and led by Halil İbrahim Kalkan. It is the .NET Core-era successor to the older ASP.NET Boilerplate project[^1], rewritten around modularity and a domain-driven-design layering convention. Where ASP.NET Core hands you a bare web host and leaves architecture to you, ABP ships a prescribed solution shape: separate `Domain`, `Application`, `EntityFrameworkCore`, `HttpApi`, and `Web` projects, wired together by a module system and a convention-based dependency-injection container. At ~14.3k stars it is one of the largest opinionated application frameworks in the .NET ecosystem, and it is actively developed — commits land on the `dev` branch continuously.

The defining tension is open-core. The framework in this repository is LGPL-3.0 and genuinely capable, but ABP is also a commercial product (the abp.io platform, formerly "ABP Commercial"). Many of the batteries-included pieces you see in marketing and tutorials — the Pro variants of Identity and Account, the SaaS/tenant-management module, ABP Suite (page generator), and much of ABP Studio — are paid[^2]. The free tier gives you the framework plus basic open-source modules; the productivity story that sells ABP largely lives behind a subscription. Evaluating ABP means being clear about which side of that line each feature falls on.

The second tension is weight. ABP's abstractions (unit-of-work interceptors, auto-generated API controllers, dynamic HTTP client proxies, a virtual file system, permission/setting/audit subsystems) remove a lot of boilerplate, but they also mean a new team is learning ABP's conventions as much as it is writing business code. The framework rewards large, long-lived enterprise applications and punishes small ones.

## Getting Started

```bash
# Install the ABP CLI (global .NET tool)
dotnet tool install -g Volo.Abp.Studio.Cli

# Create a layered solution (EF Core + MVC/Razor UI here)
abp new Acme.BookStore -u mvc -d ef

cd Acme.BookStore/aspnet-core
# apply EF Core migrations + seed the initial data/admin user
dotnet run --project src/Acme.BookStore.DbMigrator
# run the web app
dotnet run --project src/Acme.BookStore.Web
```

A minimal application service — ABP auto-exposes it as a REST API controller with no explicit controller class:

```csharp
public class BookAppService : ApplicationService, IBookAppService
{
    private readonly IRepository<Book, Guid> _books;
    public BookAppService(IRepository<Book, Guid> books) => _books = books;

    // Becomes GET /api/app/book — auto API controller convention
    public async Task<List<BookDto>> GetListAsync()
    {
        var books = await _books.GetListAsync();
        return ObjectMapper.Map<List<Book>, List<BookDto>>(books);
    }
}
```

## Architecture / How It Works

ABP is a **modular monolith** first. The unit of composition is a *module* — a class deriving from `AbpModule`, annotated with `[DependsOn(...)]`, that configures services in `ConfigureServices`. The startup application is itself just a module that depends on framework and feature modules; the dependency graph is resolved at boot. This is what lets ABP ship reusable "application modules" (Identity, Account, Tenant Management, CMS Kit, etc.) as NuGet packages that drop into a host.

The layering follows DDD: entities and domain services in `Domain`, application services and DTOs in `Application`, persistence in `EntityFrameworkCore` (or `MongoDB` — ABP's `IRepository<>` abstraction supports both), and API/UI on top. **Auto API controllers** turn application services into REST endpoints by convention, and **dynamic C# proxies** generate typed HTTP clients for those same endpoints, so a Blazor/MAUI client can call the server through the same interface the server implements.

Cross-cutting concerns are handled by **interception**. ABP uses Castle DynamicProxy to wrap services with interceptors for unit-of-work (transaction/`DbContext` scope), authorization, auditing, and validation. This is why so little of that plumbing appears in your code — and also why stack traces are deep and why some behavior is implicit rather than visible at the call site.

Other core subsystems: a **virtual file system** (embed views/assets in NuGet packages), a distributed **event bus** (in-process by default; RabbitMQ/Kafka providers for microservices), **background jobs**, **permission/setting/feature management** stores, **audit logging**, and **multi-tenancy** implemented as an ambient `ICurrentTenant` plus data filtering that scopes queries per tenant (single-DB or DB-per-tenant). Authentication is built on **OpenIddict**; ABP migrated its default from IdentityServer to OpenIddict after IdentityServer's shift to a commercial license[^3]. UI options include MVC/Razor Pages, Angular, and Blazor (Server and WebAssembly).

## Production Notes

- **Open-core is the biggest planning variable.** The free modules cover authentication, basic user/role management, and tenant management. Advanced UIs (organization units, claims, external logins, LDAP), the SaaS module, ABP Suite's CRUD page generation, and premium themes are commercial[^2]. Teams routinely discover mid-project that a demoed feature requires a subscription. Decide the licensing posture before committing.
- **Solution sprawl.** A default template generates 6–10+ projects. This is deliberate (enforces the layering) but means longer build times, more moving parts in CI, and a steeper mental map than a single ASP.NET Core project. For a small service this is overhead you pay without recouping.
- **Interception overhead and startup cost.** DynamicProxy-based unit-of-work/audit/auth interceptors add per-call indirection, and module boot resolves a large service graph. It is fine for typical enterprise line-of-business load, but ABP is not the choice for a latency-critical microservice where you want to see and control every allocation.
- **Upgrades track .NET major versions.** ABP majors align to .NET releases (ABP 8 → .NET 8, ABP 9 → .NET 9), so you upgrade roughly yearly. The CLI's `abp update` bumps the coordinated package set, but cross-major migrations can carry breaking changes; the IdentityServer→OpenIddict switch was the most disruptive historically and required manual migration of auth configuration and data.
- **EF Core migrations are modular.** Each module can contribute to the `DbContext`, and the generated `DbMigrator` project owns schema + data seeding. Running the app without first running the migrator (or without seeded permission/setting data) is a common first-run stumble.
- **Conventions can surprise.** Auto API controllers, implicit unit-of-work boundaries, and the ambient tenant filter mean behavior is often configured elsewhere than the call site. Debugging "why did this query return nothing" frequently ends at the multi-tenancy data filter or a missing `[AllowAnonymous]`.

## When to Use / When Not

**Use when:**
- You are building a long-lived enterprise/B2B application on .NET and want DDD layering, RBAC, audit logging, and multi-tenancy provided rather than assembled.
- You want a SaaS starting point (tenants, editions, subscriptions) and are willing to buy the commercial modules that complete it.
- Your team values a prescribed architecture and consistency across many developers over minimalism.

**Avoid when:**
- You are shipping a small service or an MVP where ABP's project sprawl and learning curve outweigh what it saves.
- You want to see and control every layer explicitly; the interception and conventions are the opposite of transparent.
- The features you actually need turn out to be commercial-only and the subscription doesn't fit your budget or licensing constraints.
- You need a framework with rare breaking changes; ABP moves with .NET's yearly cadence.

## Alternatives

- dotnet/aspnetcore — the unopinionated baseline; build your own architecture with no framework lock-in when ABP's conventions are more than you want.
- ardalis/CleanArchitecture — a solution template (not a framework) when you want DDD layering guidance without runtime abstractions.
- OrchardCore/OrchardCore — modular ASP.NET Core application/CMS framework when your app is content- and module-driven.
- aspnetboilerplate/aspnetboilerplate — ABP's own predecessor; only relevant for legacy .NET Framework apps, not new work.
- ServiceStack/ServiceStack — commercial full-stack .NET services framework when you want an integrated stack with a different (services-first) philosophy.

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo) | 2016-12 | `abpframework/abp` created as the ASP.NET Core rewrite of ASP.NET Boilerplate[^1]. |
| 6.0 | 2022 | Default authentication switched from IdentityServer to OpenIddict[^3]. |
| 8.0 | 2023-11 | Aligned to .NET 8. |
| 9.0 | 2024-11 | Aligned to .NET 9. |
| — | 2024 | ABP Studio (cross-platform desktop IDE for ABP solutions) becomes the primary tooling front end[^4]. |

## References

[^1]: ASP.NET Boilerplate — the predecessor project ABP Framework was rewritten from. https://aspnetboilerplate.com
[^2]: ABP pricing / commercial platform (Pro modules, ABP Suite, premium themes). https://abp.io/pricing
[^3]: ABP docs, OpenIddict integration (default auth provider after the IdentityServer migration). https://abp.io/docs/latest/framework/fundamentals/openiddict
[^4]: ABP Studio — cross-platform desktop application for ABP developers. https://abp.io/studio

## Tags

csharp, dotnet, aspnet-core, application-framework, domain-driven-design, modular-monolith, multi-tenancy, saas, open-core, enterprise, blazor
