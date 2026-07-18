# digitallyinduced/ihp

> Batteries-included Haskell MVC web framework where the database schema is the
> source of truth and Nix supplies the entire toolchain.

[GitHub repo](https://github.com/digitallyinduced/ihp) ·
[Official website](https://ihp.digitallyinduced.com/) ·
[License: MIT](https://github.com/digitallyinduced/ihp/blob/master/LICENSE)

## Overview

IHP ("Integrated Haskell Platform") is a full-stack MVC framework built by
digitally induced, a German software agency. It was used internally from 2017
and released publicly in mid-2020[^1]; v1.0 shipped in December 2022[^2]. At
~5.3k stars with pushes to master as recently as this week, it is the most
active batteries-included web framework in the Haskell ecosystem — a niche
where most competitors (Yesod, Snap) predate it by a decade and iterate far
more slowly.

The defining bet is **making Haskell usable by non-Haskellers**. IHP
deliberately hides monad transformers, avoids type-level programming in user
code, and leans on implicit parameters (`?context`, `?modelContext`) —
choices that experienced Haskellers often dislike but that keep controller
code looking like scripting. The second bet is **Nix as a hard dependency**:
GHC, PostgreSQL, and every library are pinned through devenv, giving
identical environments across a team at the cost of adopting the Nix
toolchain, its disk footprint, and its failure modes.

Recent positioning targets AI-assisted development: the framework ships a
`CLAUDE.md` teaching agents its conventions, on the argument that a strong
type system is the verification layer for generated code[^3].

## Getting Started

```bash
nix profile install nixpkgs#ihp-new
ihp-new myproject
cd myproject && devenv up   # starts GHC dev server + PostgreSQL + IDE
```

A controller action with IHP's fill/validate pipeline and HSX templating:

```haskell
action CreatePostAction = do
    let post = newRecord @Post
    post
        |> fill @'["title", "body"]
        |> validateField #title nonEmpty
        |> ifValid \case
            Left post  -> render NewView { .. }
            Right post -> do
                post <- post |> createRecord
                redirectTo PostsAction

renderPost :: Post -> Html
renderPost post = [hsx|<article><h2>{post.title}</h2><p>{post.body}</p></article>|]
```

## Architecture / How It Works

**Schema-first code generation.** `Application/Schema.sql` — plain PostgreSQL
DDL, editable via a browser-based Schema Designer GUI — is compiled into a
`Generated.Types` module containing record types, and the query builder
operates over those types with `OverloadedLabels` field references
(`#title`). There is no separate ORM mapping layer to drift from the
database; conversely, there is no supported path to any database other than
PostgreSQL.

**HSX.** A Template Haskell quasiquoter embedding JSX-like HTML in Haskell,
checked at compile time and compiled down to blaze-html[^4]. `ihp-hsx` is
published as a standalone package usable outside the framework.

**Routing by convention.** `AutoRoute` derives URL routes from controller
data-type constructors (`ShowPostAction { postId }` becomes
`/ShowPost?postId=...`), with optional custom route instances for pretty
URLs.

**Auto Refresh.** Rendering an action records which tables its queries
touched; the server then watches those tables through PostgreSQL triggers and
notifications, re-renders the view on change, and patches the client DOM
over a WebSocket using morphdom[^5]. Server-rendered views become
live-updating with one line — IHP's answer to Phoenix LiveView, with the
same implication: per-session server-side state.

**DataSync.** A TypeScript client (`ihp-datasync`) exposing CRUD directly
against Postgres for SPA-style frontends, authorized by PostgreSQL row-level
security policies declared in the schema[^6].

**Dev loop.** The dev server loads the application through the GHC bytecode
interpreter for fast reload despite Haskell being compiled; production
builds are optimized native binaries. Deployment's blessed path is
`deploy-to-nixos`, which runs `nixos-rebuild` over SSH so nginx, TLS,
Postgres, and systemd units are declared in-repo[^7].

## Production Notes

- **Nix is the tax.** First builds without the binary cache can compile large
  parts of the Haskell world; configure digitally induced's cache (and your
  own for CI) or expect hour-scale cold builds. `/nix/store` grows tens of
  GB. macOS Nix upgrades occasionally break dev environments in ways that
  have nothing to do with your code.
- **Compile times grow with the app.** Generated record types plus HSX TH
  splices make GHC memory-hungry; CI runners below ~8 GB RAM struggle on
  mid-sized apps. Dev-mode reloads stay fast; full optimized builds do not.
- **Auto Refresh state is in-process.** Horizontal scaling requires sticky
  sessions, and each live session holds server memory. Most IHP deployments
  are single-server NixOS boxes, which the tooling is honest about.
- **PostgreSQL-only** is a feature until a client mandates MySQL or a managed
  DB with restricted extensions/triggers (Auto Refresh needs to install
  triggers).
- **Upgrade friction is real.** The v1.0→v1.1 move replaced the entire dev
  environment with devenv/flakes — a workflow migration, not a code one[^2].
  The formerly managed "IHP Cloud" hosting service was discontinued, leaving
  self-managed NixOS as the primary path.
- **Hiring.** Haskell's pool is small; IHP's simplified style genuinely
  lowers the bar for onboarding, but debugging GHC errors under deadline
  still requires someone who knows the language.

## When to Use / When Not

**Use when:**
- A small team wants Rails-style productivity with compile-time guarantees
  across routing, SQL, and HTML.
- You control your infrastructure and a declarative NixOS single-server
  deployment is acceptable or attractive.
- You want live-updating server-rendered views without writing a JS frontend.

**Avoid when:**
- You cannot adopt Nix (locked-down corporate machines, unsupported CI) —
  there is no supported non-Nix development path.
- You need a database other than PostgreSQL, or a DBaaS that forbids
  triggers.
- You need a large hiring pool or expect heavy custom type-level Haskell —
  IHP's conventions fight both directions.
- You are building a pure JSON API: the framework's value is in the
  integrated server-rendered stack.

## Alternatives

- yesodweb/yesod — use instead when you want a mature, cabal/stack-native
  Haskell framework without a Nix requirement.
- haskell-servant/servant — use instead for API-first services where the
  route spec as a type is the product.
- scotty-web/scotty — use instead for small Sinatra-style Haskell services
  with minimal framework surface.
- phoenixframework/phoenix — use instead if you want the LiveView model with
  a larger community and BEAM's horizontal-scaling story.
- rails/rails — use instead when team familiarity and ecosystem breadth beat
  static guarantees.

## History

| Version | Date | Notes |
|---------|------|-------|
| Internal | 2017-12 | Repo created; used in digitally induced client work[^8]. |
| Public release | 2020-06 | Open-sourced; HN launch[^1]. |
| 1.0 | 2022-12 | First stable release[^2]. |
| 1.1 | 2023 | Dev environment rebuilt on devenv/Nix flakes[^2]. |
| master | 2026-07 | Actively developed; AI/`CLAUDE.md` workflow emphasis[^3]. |

## References

[^1]: IHP releases (first public tags, June 2020). https://github.com/digitallyinduced/ihp/releases
[^2]: IHP release notes v1.0 / v1.1. https://github.com/digitallyinduced/ihp/releases
[^3]: IHP README, "AI-Driven Development". https://github.com/digitallyinduced/ihp#readme
[^4]: HSX guide. https://ihp.digitallyinduced.com/Guide/hsx.html
[^5]: Auto Refresh guide. https://ihp.digitallyinduced.com/Guide/auto-refresh.html
[^6]: DataSync / realtime SPAs guide. https://ihp.digitallyinduced.com/Guide/realtime-spas.html
[^7]: Deployment guide. https://ihp.digitallyinduced.com/Guide/deployment.html
[^8]: GitHub repository metadata (created 2017-12-03). https://github.com/digitallyinduced/ihp

## Tags

haskell, web-framework, mvc, postgresql, nix, type-safety, server-side-rendering, code-generation, hsx, full-stack
