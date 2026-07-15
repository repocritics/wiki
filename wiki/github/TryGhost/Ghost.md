# TryGhost/Ghost

> Node.js publishing platform for professional blogs, newsletters, and paid memberships — a CMS that pivoted into a creator-subscription business in a box.

[GitHub repo](https://github.com/TryGhost/Ghost) ·
[Official website](https://ghost.org) ·
[License: MIT](https://github.com/TryGhost/Ghost/blob/main/LICENSE)

## Overview

Ghost started in 2013 as a Kickstarter-funded reaction to WordPress: a focused, Node.js publishing tool that did blogging and nothing else[^1]. It is governed by the non-profit Ghost Foundation, and the commercial Ghost(Pro) managed-hosting service funds development — 100% of that revenue goes back to the Foundation. This structure is the reason the project stays open source and the reason its roadmap is aimed at publishers rather than shareholders.

The defining shift came with Ghost 3.0 (2019), which added native **Members and subscriptions**[^2], followed by Ghost 4.0 (2021) with built-in email **newsletters, tiers, and a Portal** paywall[^3]. Ghost is no longer "an open-source blog"; it is an integrated stack for running a paid publication — content, membership, Stripe billing, and bulk newsletter delivery in one install. That integration is its strength (you do not assemble five services) and its constraint (it is opinionated toward one shape of product).

The tension to understand before adopting: Ghost is deeply pleasant to run *the way it wants to be run* — MySQL 8, ghost-cli, Mailgun for email, a single instance — and noticeably awkward the moment you step off that path. Most production pain in the Ghost ecosystem is a symptom of fighting those defaults.

## Getting Started

Install the CLI, then a local or production instance:

```bash
npm install ghost-cli -g

# Local dev instance (SQLite, no SSL) — up in under a minute
ghost install local

# Production (Ubuntu + MySQL + nginx + LetsEncrypt SSL)
ghost install
```

A minimal Handlebars theme template (`post.hbs`) rendering a single post:

```handlebars
{{!< default}}
<article class="post">
    <h1>{{title}}</h1>
    {{#if feature_image}}
        <img src="{{feature_image}}" alt="{{title}}">
    {{/if}}
    <section class="content">
        {{content}}
    </section>
    {{#if primary_author}}<span>By {{primary_author.name}}</span>{{/if}}
</article>
```

Or consume Ghost headlessly through the read-only Content API:

```bash
curl "https://demo.ghost.io/ghost/api/content/posts/?key=22444f78447824223cefc48062&limit=3"
```

## Architecture / How It Works

Ghost is a JavaScript monorepo (Nx-managed) whose core is `ghost/core`, a Node.js/Express application[^4]. The pieces that matter:

- **Data layer** — Bookshelf.js ORM over Knex query builder. SQLite is used for development; MySQL 8 is the supported production database. There is no PostgreSQL support and none is planned.
- **Admin client** — Ghost Admin is a separate Ember.js single-page app that talks to the Admin API. It is a large, long-lived Ember codebase, which is one reason the admin UI evolves conservatively.
- **Editor** — Koenig, a block/rich-text editor. Ghost migrated its document storage from Mobiledoc to **Lexical** (Meta's editor framework) over the 5.x line[^5]; both formats coexist in older content.
- **Themes** — server-rendered Handlebars (`.hbs`). Themes are validated by GScan, and the default theme is Source (Casper in earlier versions). This is classic server-side templating, not a React/JAMstack front end.
- **APIs** — a read-optimized **Content API** (for headless/static front ends) and a read-write **Admin API** (token-authenticated). Both are REST/JSON.
- **Members & billing** — a first-class subsystem wired directly to Stripe for subscriptions, tiers, and one-off payments; Portal is the embedded signup/paywall widget.
- **Email** — transactional email can use any SMTP provider, but **bulk newsletter delivery is built specifically around Mailgun**; there is no generic bulk-SMTP path.

An **ActivityPub / fediverse** integration has been under active development to let publications federate with Mastodon and other servers[^6]; treat it as newer surface area rather than long-stable core.

## Production Notes

**Database is not negotiable.** Production Ghost expects **MySQL 8**. Ghost 5 dropped the older MySQL 5.7 recommendation, and SQLite is explicitly dev-only[^7]. Attempting to run production on SQLite or an unsupported MySQL will work until it doesn't (collation, JSON, and migration edge cases).

**Newsletters mean Mailgun.** If sending newsletters to members is core to your use case, you are effectively required to use Mailgun — this surprises teams that already run SES or Postmark. Transactional mail (password resets, member magic links) is flexible; bulk send is not.

**ghost-cli is the supported path, and stepping off it hurts.** The CLI manages nginx, systemd, SSL, backups, and — critically — **upgrades** (`ghost update`). Manual/source installs are supported for contributors but are a maintenance liability for production. Docker deployments exist but are community-driven, not the blessed route.

**Scaling is vertical by default.** Ghost runs scheduling, image processing, and background jobs in-process and assumes a single instance. It is not a stateless, horizontally-scalable app you can naively put behind a load balancer; multi-instance setups require externalizing scheduling, storage (an S3-compatible storage adapter for images instead of local disk), and session/cache concerns. Ghost(Pro) exists in large part because getting this right yourself is real work.

**Node version pinning.** Ghost supports specific Node.js LTS lines and refuses to boot on unsupported versions. Check the compatibility matrix before upgrading Node, and prefer letting ghost-cli dictate the environment.

**Upgrade discipline.** Minor upgrades are smooth via the CLI. Major version jumps (e.g., 3→4→5) have carried database migrations and dependency-floor bumps (Node, MySQL); read the migration notes and take a backup — `ghost update` can roll back, but only if the pre-upgrade state was clean.

## When to Use / When Not

**Use when:**
- You are running a publication, newsletter, or paid-membership site and want billing, email, and a paywall without integrating them yourself.
- You want clean server-rendered SEO-friendly output with Handlebars themes, or a headless front end fed by the Content API.
- You value the non-profit governance and want to self-host on the supported MySQL/ghost-cli stack.

**Avoid when:**
- You need a general-purpose CMS with arbitrary content modeling and relations — Ghost's schema is publishing-shaped and not meant to be extended like Strapi or Directus.
- You are locked to PostgreSQL, a non-Mailgun bulk-email provider, or a horizontally-scaled stateless deployment model.
- You want a large plugin/marketplace ecosystem to bolt on features (forums, e-commerce, forms) — Ghost integrates via APIs and Zapier rather than in-process plugins.

## Alternatives

- WordPress/WordPress — vastly larger plugin ecosystem and general-purpose CMS; choose it when you need extensibility and content types Ghost won't model, and can accept PHP.
- strapi/strapi — headless Node.js CMS with flexible content modeling and admin; choose it when the content schema, not publishing/membership, is the point.
- directus/directus — database-first headless platform over any SQL schema; choose it when you want a data API layer rather than an opinionated publishing product.
- payloadcms/payload — TypeScript-native headless CMS/app framework; choose it when you want code-defined collections and tight Next.js integration.
- gohugoio/hugo — static site generator; choose it for a purely static blog with no members, billing, or dynamic backend.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.3 | 2013-10 | First public release after Kickstarter[^1]. |
| 1.0 | 2017-09 | Rebuilt admin, Koenig editor introduced, ghost-cli. |
| 2.0 | 2018-08 | Koenig default, dynamic routing / structured content[^8]. |
| 3.0 | 2019-10 | Native Members & subscriptions — the membership pivot[^2]. |
| 4.0 | 2021-03 | Newsletters, tiers, Portal, offers[^3]. |
| 5.0 | 2022-05 | MySQL 8 required, official custom-theme workflow[^7]. |

## References

[^1]: John O'Nolan, "Ghost has launched" — Ghost blog, 2013. https://ghost.org/about/
[^2]: Ghost 3.0 announcement — memberships and subscriptions. https://ghost.org/blog/3-0/
[^3]: Ghost 4.0 announcement — newsletters, tiers, Portal. https://ghost.org/blog/4-0/
[^4]: Ghost architecture & contributing docs. https://ghost.org/docs/contributing/
[^5]: Ghost Content API and editor (Koenig/Lexical) documentation. https://ghost.org/docs/
[^6]: Ghost ActivityPub / social web initiative. https://activitypub.ghost.org/
[^7]: Ghost 5.0 announcement and hosting requirements. https://ghost.org/blog/5-0/
[^8]: Ghost 2.0 announcement — dynamic routing. https://ghost.org/blog/2-0/

## Tags

javascript, nodejs, cms, publishing, blogging, newsletters, memberships, headless-cms, handlebars, self-hosted, mysql
