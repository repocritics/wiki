# OrchardCMS/OrchardCore

> A modular, multi-tenant application framework on ASP.NET Core, with a CMS built on top of it.

[GitHub repo](https://github.com/OrchardCMS/OrchardCore) ·
[Official website](https://orchardcore.net) ·
[License: BSD-3-Clause](https://github.com/OrchardCMS/OrchardCore/blob/main/LICENSE)

## Overview

Orchard Core is two things bundled in one repository: an application framework
for building modular, multi-tenant apps on ASP.NET Core, and a content
management system (Orchard Core CMS) assembled from modules on top of that
framework[^1]. It is a ground-up rewrite of the original Orchard CMS — an
ASP.NET MVC / .NET Framework project dating to 2010 — reworked for the
cross-platform .NET Core stack. The rewrite spent years in beta before shipping
1.0 in July 2021[^2]; the current line is 3.x.

The defining bet is composition over configuration. Almost nothing is
monolithic: features, content types, themes, and even the admin UI are modules
and content parts you assemble at runtime, per tenant. This buys extreme
flexibility — one process can serve many isolated sites, each with its own
schema, features, and database — at the cost of a steep conceptual load. The
"shape" display system, display drivers, placement files, recipes, and the
YesSql document store are all things a developer must learn before Orchard Core
feels productive. It is a framework for teams that want to build a content
platform, not a drop-in blog.

Community size is the honest caveat: relative to WordPress or even the other
major .NET CMS (Umbraco), the contributor base and third-party module ecosystem
are small. It is a .NET Foundation project[^3] with steady releases, but you are
buying into a smaller pond.

## Getting Started

```bash
# Scaffold a new CMS site from the official templates
dotnet new install OrchardCore.ProjectTemplates::3.0.1
dotnet new occms -o MySite
cd MySite
dotnet run
# then complete setup in the browser (choose recipe, database, admin user)
```

Or wire the CMS into a minimal ASP.NET Core host directly:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOrchardCms();

var app = builder.Build();
app.UseOrchardCore();
app.Run();
```

The first run serves a setup screen rather than a site: you pick a recipe
(declarative site definition), a database provider, and create the super-admin.

## Architecture / How It Works

**Modules and features.** A module is an ASP.NET Core assembly with a `Startup`
class (or `Manifest`) that Orchard discovers and composes. Modules expose
*features* that tenants enable or disable at runtime; toggling a feature can add
routes, services, content types, and admin menu entries without a redeploy.

**Multi-tenancy via shells.** Each tenant runs in an isolated *shell* — its own
dependency-injection container, feature set, and database prefix or connection.
A single OS process hosts many tenants; requests are routed to the right shell
by host/prefix. Changing a tenant's features or settings restarts that shell,
not the process.

**YesSql persistence.** Content is not stored in hand-designed relational
tables. Orchard Core uses YesSql[^4], a document library by the project's lead
that stores objects as JSON documents in a relational database (SQL Server,
PostgreSQL, MySQL, SQLite) and maintains queryable *map/reduce index* tables
that you define with index providers. This is the single most consequential
design decision: it makes the content model dynamic and schema-light, but means
ad-hoc SQL reporting against content is awkward, and every efficient query path
must be backed by an index you wrote.

**Composable content model.** A Content Item is composed of Content Parts and
Content Fields; Content Types are defined at runtime (often through the admin UI
or a recipe) rather than as compiled classes. Rendering goes through *shapes* —
dynamic view models — resolved by *display drivers* and `placement.json`, a
templating indirection inherited conceptually from Orchard 1. Views are written
in Razor or Liquid (via the Fluid library).

**Recipes and migrations.** Recipes are JSON files that declare setup steps,
content, and configuration — config-as-code for a site. Deployment plans move
content and settings between environments. Per-module `DataMigration` classes
run on shell activation to evolve index schemas.

The CMS is just the set of modules (content management, admin, themes, media,
GraphQL, workflows, etc.) layered on the framework; you can use the framework
alone to build non-CMS modular apps, or run the CMS in decoupled/headless mode
and consume content over the GraphQL and content APIs.

## Production Notes

- **Design your indexes up front.** Because content lives as JSON documents in
  YesSql, list/filter/sort performance depends on map/reduce index providers.
  Retrofitting an index onto a large content set means a migration that reindexes
  existing documents — plan it before the data grows.
- **Tenants cost memory.** Each active shell loads its own module graph and DI
  container. Hosting many tenants in one process scales operationally but grows
  the memory footprint roughly with active-tenant count; idle tenants can be
  configured to unload.
- **Rendering can be CPU-bound.** The shape/display-driver pipeline is flexible
  but not free. Production sites lean on the built-in dynamic cache and
  fragment/output caching; an uncached content-heavy page can be surprisingly
  expensive to render.
- **The learning curve is the real cost.** Placement, shapes, display drivers,
  and drivers-vs-handlers trip up newcomers. Budget onboarding time; the docs are
  thorough but the mental model is large.
- **.NET version coupling.** Each major Orchard Core version tracks a recent
  .NET release — 2.0 (2024) required .NET 8[^5]. Upgrading Orchard Core usually
  means upgrading the target framework and any modules in lockstep.
- **Security reports go private.** The project asks that vulnerabilities be
  emailed to contact@orchardcore.net rather than filed as public issues[^1].

## When to Use / When Not

**Use when:**
- You are a .NET shop and want a content platform you can extend in C# and Razor.
- You need genuine multi-tenancy — many isolated sites from one deployment.
- Your content model is complex or evolving and you want to define types at
  runtime instead of migrating tables.
- You want config-as-code (recipes) and headless/GraphQL delivery in one stack.

**Avoid when:**
- You want a five-minute blog or marketing site — WordPress or a static-site
  generator is far less to learn.
- Your team is not on .NET; the whole model assumes ASP.NET Core hosting.
- You need a large marketplace of ready-made plugins and themes; the ecosystem
  is smaller than WordPress's or Umbraco's.
- You rely on ad-hoc SQL reporting directly against content tables.

## Alternatives

- umbraco/Umbraco-CMS — the other major .NET CMS; use it when you want a larger
  community and a more conventional relational content model over pure modularity.
- PiranhaCMS/piranha.core — use it when you want a lightweight .NET Core CMS
  without Orchard's shape/module depth.
- Squidex/squidex — use it when you want a .NET headless, event-sourced CMS
  focused purely on API delivery.
- strapi/strapi — use it when you are on Node.js and want a headless CMS with a
  broad plugin ecosystem instead of a .NET stack.
- OrchardCMS/Orchard — the legacy .NET Framework predecessor; only for
  maintaining existing Orchard 1.x sites, not new builds.

## History

| Version | Date | Notes |
|---------|------|-------|
| (original Orchard) | 2010–2011 | ASP.NET MVC / .NET Framework CMS; predecessor project. |
| 1.0.0 | 2021-07-22 | First GA of the ASP.NET Core rewrite after a long beta[^2]. |
| 1.4.0 | 2022-06-01 | Continued 1.x line on .NET 6-era tooling. |
| 1.8.0 | 2024-01-02 | Final 1.x feature line. |
| 2.0.0 | 2024-09-09 | Major release; moved to .NET 8[^5]. |
| 2.1.0 | 2024-11-21 | First 2.x maintenance/feature update. |
| 3.0.0 | 2026-06-18 | Current major line. |
| 3.0.1 | 2026-07-09 | Latest patch at time of writing[^6]. |

## References

[^1]: Orchard Core README. https://github.com/OrchardCMS/OrchardCore
[^2]: Orchard Core 1.0.0 release. https://github.com/OrchardCMS/OrchardCore/releases/tag/v1.0.0
[^3]: .NET Foundation project listing. https://dotnetfoundation.org/projects
[^4]: YesSql — document database library used by Orchard Core. https://github.com/sebastienros/yessql
[^5]: Orchard Core 2.0.0 release. https://github.com/OrchardCMS/OrchardCore/releases/tag/v2.0.0
[^6]: Orchard Core releases. https://github.com/OrchardCMS/OrchardCore/releases

## Tags

csharp, dotnet, aspnet-core, cms, content-management, modular, multi-tenant, headless-cms, framework, yessql, graphql
