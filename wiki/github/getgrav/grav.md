# getgrav/grav

> A flat-file CMS in PHP: content lives as Markdown files with YAML front matter, rendered through Twig — no database.

[GitHub repo](https://github.com/getgrav/grav) ·
[Official website](https://getgrav.org) ·
[License: MIT](https://github.com/getgrav/grav/blob/develop/LICENSE.txt)

## Overview

Grav is a database-free content management system written in PHP. Instead of storing pages in MySQL, it keeps each page as a folder on disk containing a Markdown file with a YAML front-matter header; the directory tree *is* the content tree and the URL structure[^1]. Rendering goes through Twig templates, configuration is YAML, and the whole runtime is assembled from established Symfony components (Event Dispatcher, Console, Cache) plus a Pimple dependency-injection container[^2]. First released around 2016, it sits between two worlds: it is more dynamic than a static site generator (there is a running PHP process, an admin panel, forms, and plugins) but lighter to deploy than a database CMS (unzip and go, no schema, no migrations).

The defining tradeoff is the flat-file model itself. Removing the database removes an entire class of operational burden — no DB server to provision, back up, or secure; the whole site is a directory you can copy, `git`-track, or diff. In exchange, every read that a database would index becomes a filesystem walk. Grav compensates with aggressive caching (a page cache, a compiled-Twig cache, and an image cache), which means the flat-file simplicity is real for authoring but the performance story depends heavily on caches being warm and OPcache being enabled.

Grav's audience is developers and agencies building small-to-medium content sites — documentation portals, marketing sites, blogs, brochureware — who want templating control and version-controllable content without standing up WordPress. It is a poor fit for large editorial teams or high-write, data-heavy applications.

## Getting Started

Requires PHP 8.3 or higher[^3]. The fastest path is the pre-built package from getgrav.org/downloads (zero install — extract and serve). Via Composer:

```bash
composer create-project getgrav/grav ~/webroot/grav
```

Or from source, where you must run the CLI to pull theme/plugin dependencies:

```bash
git clone https://github.com/getgrav/grav.git ~/webroot/grav
cd ~/webroot/grav
bin/grav install        # installs the default Antimatter/Quark theme + error/problems plugins
```

A page is just a directory and a Markdown file. `user/pages/01.home/default.md`:

```markdown
---
title: Home
menu: Home
---
# Welcome

This is **flat-file** content. No database was harmed.
```

The numeric prefix (`01.`) controls ordering and is stripped from the URL; the filename (`default.md`) selects the Twig template (`default.html.twig`). Install plugins/themes through the Grav Package Manager:

```bash
bin/gpm index                 # list available packages
bin/gpm install admin         # the web admin panel is itself a plugin
bin/gpm selfupgrade           # update Grav core (within a major version)
```

## Architecture / How It Works

Grav boots a single service container (`Grav\Common\Grav`, a Pimple container) that lazily wires the pages object, the cache, the Twig environment, the assets pipeline, and the plugin/event system[^2]. A request flows roughly: initialize → fire lifecycle events → resolve the current page from the filesystem tree → render its Markdown to HTML (via Parsedown) → wrap it in a Twig template → emit.

- **Pages.** The `user/pages` directory tree is scanned into an in-memory page collection. Front matter becomes page metadata; the folder hierarchy becomes routing and menu structure. Because this is a filesystem scan, the page cache is what keeps large trees usable.
- **Plugins & events.** Extensibility is entirely event-driven. Plugins subscribe to lifecycle hooks (`onPluginsInitialized`, `onPageInitialized`, `onTwigSiteVariables`, etc.) via the Symfony Event Dispatcher. Core stays small; almost everything user-facing — the admin panel, forms, SEO, login — is a plugin pulled from GPM.
- **Themes & Twig.** Themes are Twig template sets. Template resolution follows the page's filename and layout front matter. Twig is compiled and cached to disk on first render.
- **Caching.** There are multiple, independent cache layers: the page/data cache (Symfony Cache, backed by file/APCu/Redis/Memcached), the compiled-Twig cache, the asset pipeline (concatenated/minified CSS/JS), and the image cache (resized derivatives written into the `images/` folder). Understanding *which* cache is stale is the single most common source of "my change didn't show up."
- **Flex Objects.** Introduced in the 1.7 line, Flex Objects are a framework for modeling collections of structured data (users, contacts, arbitrary records) on top of flat files with indexing, so directory-like data does not degrade linearly. This is Grav's answer to the flat-file model's weakest case.

The `develop` branch is the default branch, following a git-flow model[^1] — meaning the repository's default view is the integration branch, not the last tagged release.

## Production Notes

**Caching is not optional at scale.** With caches warm and OPcache on, page delivery is fast; cold, every request walks and parses the page tree. Sites with thousands of pages feel the filesystem scan directly. Budget for a persistent cache driver (Redis/APCu rather than the default file cache) on anything non-trivial, and expect the first request after a `cache clear` to be slow.

**Multi-server deployment is awkward.** All state — cache, uploaded media, user accounts, form submissions — lives on the local filesystem. Horizontal scaling behind a load balancer means sharing or syncing that directory (NFS, object storage shims), which fights the "it's just files" simplicity. Grav's natural home is a single well-provisioned host.

**Admin panel is the security surface.** The admin is a plugin with a login. Because content and config are files, a compromised admin or a misconfigured web server that serves `.md`/`.yaml`/`user/` directly can leak configuration and accounts. Keep the admin plugin and core updated, and ensure the shipped `.htaccess`/nginx rules that block direct access to system paths are actually in effect — on nginx especially, the rewrite rules must be ported manually.

**Upgrades within vs. across major versions differ sharply.** `bin/gpm selfupgrade` handles point and minor upgrades cleanly within a major line. Crossing the 1.x → 2.0 boundary is a dedicated migration (PHP 8.3+ baseline, modernized core) and explicitly *not* covered by `selfupgrade` alone — there is a separate migration guide, and skipping it will break the site[^4].

**Git-managed installs and GPM overlap.** Many teams track `user/` in git and install core via GPM. Because GPM updates files in place, an update can conflict with git-managed files or third-party plugins that lag on the PHP/API baseline. Pin plugin versions and test upgrades on a copy.

**Not every plugin is maintained.** The GPM catalog is large but long-tailed; some plugins have gone stale relative to current PHP and Grav APIs. Check last-release dates before depending on one.

## When to Use / When Not

**Use when:**
- You want version-controllable, database-free content with real templating (Twig) and a running CMS (admin, forms, plugins) — more than a static generator, less than WordPress.
- The site is content-centric and mostly read: docs, marketing, blogs, small brochure sites.
- You value trivial deployment and backup (copy a directory) and want to keep content in git.
- Your team is comfortable in PHP/Twig/YAML.

**Avoid when:**
- You have high write volume or large relational data — the flat-file model and filesystem scans work against you (Flex Objects mitigate but don't erase this).
- You need horizontal scaling across many stateless nodes; shared-filesystem state complicates it.
- Your team wants a purely static output with no PHP runtime to secure — a static site generator is a smaller attack surface.
- You need the vast plugin/theme marketplace and hiring pool of WordPress.

## Alternatives

- statamic/cms — flat-file (or database) CMS on Laravel with a polished control panel; commercial license, heavier stack, stronger editorial UX.
- getkirby/kirby — file-based PHP CMS with a strong panel and API; commercial license, comparable philosophy, arguably nicer DX.
- picocms/Pico — much smaller, dependency-light flat-file CMS; use when Grav feels like too much machinery.
- gohugoio/hugo or 11ty/eleventy — static site generators; use when you want zero runtime and can accept build-time-only content and no admin panel.
- WordPress — database CMS; use when you need the largest plugin/theme ecosystem, non-technical editors, and easy hiring, and don't mind running MySQL.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-08 | Project public on GitHub; flat-file architecture, Twig + Markdown[^1]. |
| 1.0 | 2016 | First stable major release. |
| 1.6 | 2019 | Multi-language and performance work; PHP baseline raised over the line. |
| 1.7 | 2021-01 | Flex Objects framework, stronger typing, modernized admin/forms[^5]. |
| 2.0 | 2026 | Major release: PHP 8.3+ baseline, modernized core, dedicated 1.x→2.x migration[^4]. |

## References

[^1]: Grav README and project overview — architecture, technologies, git-flow branching. https://github.com/getgrav/grav
[^2]: Grav uses Pimple (DI container), Symfony Event Dispatcher, Symfony Console, and Symfony Cache as core components, per the README technology list. https://github.com/getgrav/grav
[^3]: Grav requirements — PHP 8.3 or higher. https://learn.getgrav.org/basics/requirements
[^4]: Migrating from Grav 1.x to Grav 2.0 (dedicated migration; `selfupgrade` alone does not cross the boundary). https://getgrav.org/migrate-to-2
[^5]: Upgrading to Grav 1.7 guide (Flex Objects line). https://learn.getgrav.org/16/advanced/grav-development/grav-17-upgrade-guide

## Tags

php, cms, flat-file, content-management, markdown, twig, yaml, symfony, static-adjacent, self-hosted
