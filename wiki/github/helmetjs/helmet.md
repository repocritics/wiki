# helmetjs/helmet

> Express middleware that sets security-related HTTP response headers so you don't hand-roll them yourself.

[GitHub repo](https://github.com/helmetjs/helmet) ·
[Official website](https://helmet.js.org/) ·
[License: MIT](https://github.com/helmetjs/helmet/blob/main/LICENSE)

## Overview

Helmet is a small collection of Express/Connect-style middleware functions, each of which sets (or removes) one security-related HTTP response header. Calling `app.use(helmet())` applies a curated default bundle — roughly a dozen headers including `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, and `Referrer-Policy`, plus removal of Express's `X-Powered-By`[^1]. It has been a near-default dependency in Express apps since the early 2010s (the repo dates to 2012[^2]) and is one of the most widely installed security packages in the Node ecosystem.

The defining property of Helmet is that it does almost nothing you couldn't do with `res.setHeader` yourself. Its value is curation and defaults: it encodes a maintained opinion about which headers matter, what safe values are, and which legacy headers should now be *disabled* rather than set (for example it ships `X-XSS-Protection: 0` to turn off the browser's buggy legacy XSS filter[^1]). That curation is also its limit — Helmet sets headers, and header-based defenses are only one layer. It does not sanitize input, escape output, validate auth, or protect against anything the browser doesn't enforce via a header.

The central tension is `Content-Security-Policy`. Helmet ships a conservative default CSP, but a real policy almost always needs per-app tuning (nonces, allowed CDNs, inline-script decisions), and Helmet deliberately performs almost no validation of what you pass — it will happily serialize an insecure policy[^1]. Teams that `app.use(helmet())` and move on often ship a CSP that either breaks their frontend or provides little protection.

## Getting Started

```bash
npm install helmet
```

```javascript
import express from "express";
import helmet from "helmet";

const app = express();

// Applies the default header bundle to every response.
app.use(helmet());

// Tune one header while keeping the rest of the defaults:
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        "script-src": ["'self'", "cdn.example.com"],
      },
    },
    // Disable a header entirely:
    xDownloadOptions: false,
  }),
);

app.listen(3000);
```

Each middleware is also usable standalone, e.g. `app.use(helmet.contentSecurityPolicy())`, which is useful when you want to apply different headers to different routers.

## Architecture / How It Works

Helmet is a thin dispatcher over a set of independent header modules. `helmet(options)` returns a single Express middleware that, per request, invokes each enabled sub-middleware in sequence; each one calls `res.setHeader` (or `res.removeHeader` for `X-Powered-By`) with its configured value and calls `next()`. There is no shared state, no request inspection beyond what CSP directive-functions opt into, and no response-body handling — Helmet only touches headers, and only at the point it runs in the middleware chain.

The public surface is a map from camelCased option keys (`contentSecurityPolicy`, `strictTransportSecurity`, `referrerPolicy`, `xFrameOptions`, …) to per-header config objects. Setting a key to `false` disables that header; omitting it uses the default; passing an object configures it. Recent major versions renamed these option keys to mirror the actual header names (e.g. the old `hsts` / `frameguard` / `noSniff` became `strictTransportSecurity` / `xFrameOptions` / `xContentTypeOptions`), which is the single most disruptive change across Helmet upgrades[^3].

CSP is the one module with real logic: directive values can be strings *or functions* `(req, res) => string`, evaluated per request, which is how per-request nonces work. Directives are merged over a built-in default set unless `useDefaults: false` is passed. Helmet is written in TypeScript, ships its own type declarations, and has zero runtime dependencies[^1] — a deliberate posture for a package that sits in the security path of many apps.

Because each header is order-independent and stateless, Helmet is genuinely framework-light: it targets the Express/Connect `(req, res, next)` signature and nothing else. Non-Express frameworks are served by separate wrapper packages (`koa-helmet`, `@fastify/helmet`) that reuse the same header logic.

## Production Notes

- **`helmet()` alone is not "secured."** The defaults are safe *headers*, not a safe *app*. CSP in particular usually needs work; the shipped default (`script-src 'self'`, etc.) will block inline scripts and third-party assets, so an unconfigured CSP frequently breaks real frontends on first deploy.
- **HSTS and `upgrade-insecure-requests` fight localhost.** `Strict-Transport-Security` and the CSP `upgrade-insecure-requests` directive cause browsers (notably Safari) to force `http://localhost` to `https://localhost`, which breaks local dev in confusing, cached ways. Disable both in development[^1].
- **HSTS is sticky.** Browsers cache `Strict-Transport-Security` for `max-age` seconds (default 365 days). Shipping it — especially with `preload` — to a domain you can't guarantee will stay on HTTPS is a long-lived footgun; `preload` is effectively irreversible on the timescale of browser preload-list updates.
- **Helmet does not validate your CSP.** It serializes whatever directives you give it. Use an external checker such as Google's CSP Evaluator; don't assume a policy Helmet accepted is a policy that protects you[^1].
- **Middleware order matters.** Helmet only sets headers on responses that flow through it, and only if placed before the handlers/routers that send them. Mounting it after a route that already responded silently sets nothing for that route.
- **Legacy headers are intentionally minimal or off.** `X-XSS-Protection` is set to `0` on purpose, `X-Frame-Options` is retained mainly for old browsers (superseded by CSP `frame-ancestors`), and `Cross-Origin-Embedder-Policy` is *not* enabled by default because it commonly breaks cross-origin resource loading[^1].
- **Upgrades can silently drop headers.** Because major versions have renamed option keys and changed the default set, an upgrade where you passed now-removed/renamed keys can quietly stop emitting a header you relied on. Diff response headers before and after a Helmet major bump[^3].

## When to Use / When Not

**Use when:**
- You run Express (or Connect) and want maintained, sensible defaults for security headers without tracking each header's spec churn yourself.
- You want per-header, per-router control with a single small dependency and no runtime deps.
- You want a real CSP surface with per-request nonce support.

**Avoid / don't rely on it alone when:**
- You expect header middleware to be a complete security strategy — it is one layer among input validation, authn/z, output encoding, and dependency hygiene.
- You're not on Express: use `@fastify/helmet`, `koa-helmet`, or your framework's native header support instead of wrapping Helmet directly.
- You terminate TLS/headers at a reverse proxy (nginx, a CDN, an API gateway) and would rather manage headers there in one place.

## Alternatives

- fastify/fastify-helmet (`@fastify/helmet`) — the same header logic wired for Fastify's hook system; use when your server is Fastify, not Express.
- venables/koa-helmet — Helmet's headers adapted to Koa's middleware model; use for Koa apps.
- Reverse-proxy header config (nginx `add_header`, Caddy, or a CDN's rules) — use when you'd rather set headers at the edge for all upstreams in one place.
- expressjs/cors — orthogonal, not a replacement: use it *alongside* Helmet when you need `Access-Control-*` CORS headers, which Helmet does not manage.
- OWASP Secure Headers project references — use when you want the underlying header guidance and intend to set headers by hand rather than via a package.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2012 | Repo created; header-middleware collection for Express[^2]. |
| 4.x | ~2020 | Trimmed the default bundle; CSP handling consolidated. |
| 5.x | ~2022 | Rewritten in TypeScript; ships own types; zero runtime deps[^1]. |
| 6.x | ~2022–2023 | Continued default-set and CSP-default refinements. |
| 7.x | ~2023 | Option keys renamed to mirror header names (`hsts`→`strictTransportSecurity`, etc.)[^3]. |
| 8.x | ~2024 | Current major line; Node/Express baseline raised. |

## References

[^1]: helmetjs/helmet README — default headers, CSP configuration, HSTS/localhost caveat, `X-XSS-Protection: 0`, zero-dependency and TypeScript posture. https://github.com/helmetjs/helmet
[^2]: GitHub API repository metadata for helmetjs/helmet — `created_at` 2012-02-01, license MIT, ~10.7k stars, primary language TypeScript. https://api.github.com/repos/helmetjs/helmet
[^3]: Helmet changelog / release notes — option-key renaming and default-set changes across major versions. https://github.com/helmetjs/helmet/blob/main/CHANGELOG.md

## Tags

javascript, typescript, nodejs, express, middleware, security, http-headers, csp, web-security, backend
