# WordPress/WordPress

> The PHP/MySQL CMS that runs a large fraction of the public web — and a read-only Git mirror, not where you actually contribute.

[GitHub repo](https://github.com/WordPress/WordPress) ·
[Official website](https://wordpress.org) ·
[License: GPL-2.0-or-later](https://github.com/WordPress/WordPress/blob/master/license.txt)

## Overview

WordPress is a self-hosted content management system written in PHP with a MySQL/MariaDB backend. It began in 2003 as a fork of the abandoned b2/cafélog blogging tool by Matt Mullenweg and Mike Little, and grew from a blog engine into a general-purpose CMS that powers a large share of all websites — commonly cited around 40% via W3Techs surveys[^1]. Its reach is the single most important fact about it: WordPress is less a "framework you choose" than an ecosystem you inherit, with tens of thousands of plugins and themes and a hosting industry built around it.

The defining tension is **backward compatibility versus architectural age**. WordPress treats not-breaking-existing-sites as a near-sacred commitment: code written against the plugin API a decade ago still tends to run[^2]. That discipline is why the ecosystem is so deep and why upgrades rarely brick a site. It is also why the core carries procedural PHP patterns, a global `$wpdb`, and a hook system that predates modern PHP — the price of never leaving anyone behind.

A second thing to understand immediately: **this GitHub repository is a mirror**. Its own description tells you not to send pull requests here. Canonical development happens in Subversion, tracked at core.trac.wordpress.org, with a Git development mirror at WordPress/wordpress-develop that does accept PRs[^3]. The `WordPress/WordPress` repo is the built, ready-to-deploy tree — useful for reading source or vendoring, not for contributing.

## Getting Started

The historically advertised "famous 5-minute install": unzip, upload, point a browser at `wp-admin/install.php`, and fill in database credentials[^4]. For local development the common path is WP-CLI:

```bash
# Download core, create config, and install — via WP-CLI
wp core download
wp config create --dbname=wp --dbuser=root --dbpass=secret
wp core install --url=localhost:8080 --title="My Site" \
  --admin_user=admin --admin_password=changeme --admin_email=you@example.com
```

A minimal plugin — the real extension unit — hooks into core via actions and filters:

```php
<?php
/**
 * Plugin Name: Hello Footnote
 */
add_filter( 'the_content', function ( $content ) {
    if ( is_single() ) {
        $content .= '<p><em>Thanks for reading.</em></p>';
    }
    return $content;
} );
```

System requirements: PHP 7.4+ (8.3+ recommended), MySQL 5.5.5+ or MariaDB 10.11+[^4].

## Architecture / How It Works

WordPress is a request-lifecycle engine wrapped around a hook system:

1. **Bootstrap** — `wp-load.php` → `wp-settings.php` loads core, the active theme, and every active plugin on every request. There is no compiled kernel; PHP re-includes the world per request (opcode caches and object caches mitigate this).
2. **The main query** — WordPress parses the URL into query variables, runs `WP_Query` against the database, and populates the global `$wp_query`.
3. **The Loop** — templates iterate `have_posts()` / `the_post()` to render results. Template selection follows the theme hierarchy (`single.php`, `page.php`, `archive.php`, …).

**Hooks** are the extensibility contract: *actions* (`do_action`) fire at lifecycle points, *filters* (`apply_filters`) let code rewrite values in flight. Nearly all customization — plugins, themes, integrations — is registered callbacks on these hooks. This is powerful and unbounded: any plugin can filter almost anything, which is both the source of the ecosystem's flexibility and its debugging pain.

**Data model.** A small, stable schema: `wp_posts` (posts, pages, and every custom post type), `wp_postmeta` (key-value metadata via the EAV pattern), `wp_options`, `wp_users`/`wp_usermeta`, `wp_terms` for taxonomies. The postmeta EAV table is flexible but a frequent performance liability at scale.

**The block editor (Gutenberg).** Since WordPress 5.0 (2018) the default editor is a React-based block editor; content is stored as HTML with block delimiter comments[^5]. Full Site Editing and block themes (5.9, 2022) extended blocks from post content to headers, footers, and templates via `theme.json`[^6]. This bifurcates the ecosystem into "classic" (PHP templates + shortcodes) and "block" (React + JSON) mental models that coexist uneasily.

**REST API.** A JSON REST API landed in core in 4.7 (2016), making WordPress usable as a headless backend[^7]. The admin (`wp-admin`) still relies heavily on `admin-ajax.php` for legacy paths.

## Production Notes

**Security surface is plugins, not core.** Core WordPress has a mature security team and a strong record; the overwhelming majority of real-world compromises come from third-party plugins and themes, out-of-date installs, and weak credentials. Treat every plugin as attack surface. Auto-updates for core (since 3.7) and, optionally, for plugins are the single highest-leverage defense.

**Performance.** The per-request "load everything" model means an uncached WordPress site is database- and PHP-bound. Standard production stack: a persistent object cache (Redis/Memcached), a full-page cache (Varnish, or a caching plugin), and PHP opcache. `wp_postmeta` and `wp_options` (especially autoloaded options) are the usual hotspots — an oversized `autoload` set silently taxes every request.

**Scaling.** WordPress scales horizontally behind a load balancer, but shared writable state lives in `wp-content/uploads` — multi-server setups need shared/object storage (S3-backed media) rather than local disk. `WP_Multisite` runs many sites from one install and one database, with real operational sharpness (schema-per-site tables grow linearly).

**Upgrade pains.**
- **Gutenberg migration** — sites and plugins built on the classic editor's `the_content` filters and shortcodes need real work to become block-native; the Classic Editor plugin is the common stall.
- **PHP version bumps** — old plugins that used removed PHP functions break on host upgrades; core itself keeps a low minimum longer than most modern projects.
- **Plugin conflicts** — because any plugin can filter anything, upgrades occasionally produce action-priority or hook-ordering regressions that are hard to bisect.

**Governance risk.** WordPress the software is GPL and community-built, but its infrastructure (WordPress.org, the plugin/theme directories, trademarks) is controlled by Matt Mullenweg and Automattic. The 2024 dispute between Mullenweg and the host WP Engine — including access changes to WordPress.org resources — is a live reminder that "open source" here does not mean "neutrally governed"[^8]. Operators depending on the .org update and plugin infrastructure should understand it is not a neutral utility.

## When to Use / When Not

**Use when:**
- You need a content/publishing site editable by non-developers, fast.
- You want an enormous plugin/theme ecosystem and a deep hiring pool.
- You need proven backward compatibility and long-lived, low-maintenance sites.
- You want either a traditional themed site or a headless REST/GraphQL backend.

**Avoid when:**
- You're building an app, not a content site — the request-per-page CMS model fights you.
- You want a modern typed codebase; core is procedural PHP with heavy globals.
- You can't commit to the security discipline plugins demand (updates, auditing, hardening).
- You want vendor-neutral governance and no dependency on one company's .org infrastructure.

## Alternatives

- getgrav/grav or a static site generator (Hugo, Jekyll) — use instead when content is developer-managed and you don't need a database or admin UI.
- TryGhost/Ghost — use instead for a focused publishing/newsletter product with a modern Node stack and no plugin sprawl.
- drupal/drupal — use instead when you need structured content modeling and fine-grained permissions out of the box.
- strapi/strapi or directus/directus — use instead when you want a headless, API-first CMS with a modern JS/TS backend.
- joomla/joomla-cms — use instead for a general-purpose PHP CMS with more built-in structure than WordPress and a smaller plugin surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.70 | 2003-05-27 | First release; fork of b2/cafélog[^1]. |
| 1.0 | 2004-01 | "Davis" — permalinks, moderation, post preview. |
| 1.5 | 2005-02 | Theme system, pages. |
| 2.7 | 2008-12 | "Coltrane" — admin redesign, one-click plugin install. |
| 3.0 | 2010-06 | Custom post types, multisite (MU merge), custom menus. |
| 3.7 | 2013-10 | Automatic background updates. |
| 4.7 | 2016-12 | REST API in core[^7]. |
| 5.0 | 2018-12 | "Bebo" — Gutenberg block editor default[^5]. |
| 5.9 | 2022-01 | "Josephine" — Full Site Editing, block themes[^6]. |
| 6.x | 2022– | Ongoing block-editor refinement; PHP 8 compatibility work. |

## References

[^1]: W3Techs, "Usage statistics of content management systems." https://w3techs.com/technologies/overview/content_management
[^2]: WordPress Core Handbook, "Backward Compatibility." https://developer.wordpress.org/apis/handbook/
[^3]: Contributing note in this repo's description; canonical dev at https://github.com/WordPress/wordpress-develop and https://core.trac.wordpress.org/
[^4]: WordPress installation readme (bundled in this repo). https://wordpress.org/documentation/article/how-to-install-wordpress/
[^5]: WordPress 5.0 "Bebo" release, 2018-12-06. https://wordpress.org/news/2018/12/bebo/
[^6]: WordPress 5.9 "Josephine" release, 2022-01-25. https://wordpress.org/news/2022/01/josephine/
[^7]: WordPress 4.7 "Vaughan" release, 2016-12-06. https://wordpress.org/news/2016/12/vaughan/
[^8]: Reporting on the 2024 Automattic–WP Engine dispute. https://wordpress.org/news/

## Tags

php, cms, mysql, blogging, content-management, gutenberg, plugins, rest-api, gpl, self-hosted
