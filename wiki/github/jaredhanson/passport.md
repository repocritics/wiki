# jaredhanson/passport

> Express-compatible authentication middleware for Node.js — a thin request-authentication core with a pluggable strategy ecosystem.

[GitHub repo](https://github.com/jaredhanson/passport) ·
[Official website](https://www.passportjs.org) ·
[License: MIT](https://github.com/jaredhanson/passport/blob/master/LICENSE)

## Overview

Passport is authentication middleware for Node.js, first released in 2011[^1]. Its stated scope is deliberately narrow: it authenticates a request and nothing else. It does not mount routes, does not assume a database schema, does not manage sessions on its own, and does not do authorization. Everything protocol-specific lives in separate npm packages called *strategies* — the project advertises over 480 of them[^2], covering local username/password, OAuth 1.0/2.0, OpenID Connect, SAML, and per-provider integrations (Google, GitHub, Facebook, Azure AD, and hundreds more).

The design tradeoff that defines Passport is minimalism versus assembly. The core is small and stable enough that it has effectively been feature-complete for years, but a working login flow requires the developer to wire together several independent pieces: a strategy package, a session store (`express-session`), serialize/deserialize callbacks, and the route handlers themselves. Passport gives you the seam; you supply the parts. This is why it remains the default answer for "auth on Express" while newer, more integrated frameworks win greenfield projects.

As of 2026 the repository is best read as mature and in maintenance mode rather than actively evolving — the last substantive push to the core was in 2024[^3], and most churn happens in the strategy packages, not here. That stability is a feature for existing deployments and a caution for new ones expecting an all-in-one solution.

## Getting Started

```bash
npm install passport passport-local express-session
```

```javascript
const express = require('express');
const session = require('express-session');
const passport = require('passport');
const LocalStrategy = require('passport-local');

passport.use(new LocalStrategy((username, password, done) => {
  User.findOne({ username }, (err, user) => {
    if (err) return done(err);
    if (!user || !user.verifyPassword(password)) {
      return done(null, false, { message: 'Invalid credentials' });
    }
    return done(null, user);
  });
}));

// What lands in the session cookie, and how to rehydrate req.user
passport.serializeUser((user, done) => done(null, user.id));
passport.deserializeUser((id, done) => User.findById(id, done));

const app = express();
app.use(session({ secret: 'keyboard cat', resave: false, saveUninitialized: false }));
app.use(passport.initialize());
app.use(passport.session());

app.post('/login',
  passport.authenticate('local', { failureRedirect: '/login' }),
  (req, res) => res.redirect('/'));       // req.user is now populated
```

## Architecture / How It Works

Passport's core is a small state machine wrapped around the Connect/Express middleware contract. Three moving parts matter:

1. **Strategies.** A strategy is an object with an `authenticate(req, options)` method. It inspects the request, then signals the outcome by calling back into Passport — `this.success(user)`, `this.fail()`, `this.redirect(url)`, `this.error(err)`, or `this.pass()`. The verify callback you pass to `new Strategy(...)` is what actually checks credentials and hands back a user (or `false`). This inversion — strategy handles the *protocol*, your callback handles the *identity lookup* — is the whole abstraction.

2. **`authenticate(name, options)`** returns Express route middleware. It instantiates the named strategy against the current request and dispatches on the outcome: attach `req.user` and continue, redirect, or 401. It also supports a custom-callback form (`authenticate('local', (err, user, info) => ...)`) that hands control back to you instead of applying the default behavior — commonly needed for APIs that return JSON rather than redirects.

3. **Session integration.** Passport does not own the session. It piggybacks on `express-session`. `serializeUser` decides what minimal token (usually a user ID) goes into the session; `deserializeUser` runs on **every authenticated request** to turn that token back into `req.user`. `passport.session()` is itself implemented as a strategy (`session`) that reads the serialized value.

The coupling story is the important part: Passport is bound to the Connect middleware signature (`req, res, next`) and to `express-session`'s API surface, not to any particular protocol. That is why it survived Express 3 → 4 → 5 with minor changes, but also why it feels foreign in non-Express runtimes (Fastify, Koa, serverless handlers) where the middleware and session assumptions do not hold.

## Production Notes

- **`deserializeUser` is a per-request cost.** By default it fires on every request carrying a session, meaning a database round-trip per hit unless you cache. On busy apps this is the first thing to profile and cache (in-memory LRU or Redis), or slim down by storing more of the user in the session token.

- **The 0.6.0 session-fixation change broke logout.** To fix a session-fixation weakness, `req.logout()` became asynchronous and now **requires a callback** (`req.logout(err => ...)`); the old synchronous `req.logout()` silently stopped completing[^4]. Session regeneration on login was also tightened. Upgrading across 0.5 → 0.6 without touching logout code is a known footgun that leaves users appearing logged in.

- **Callback-style verify functions.** The core and most first-party strategies predate `async/await` and expect the Node error-first `done(err, user, info)` signature. Mixing an `async` verify function without invoking `done` (or without returning correctly) leads to hung requests. Wrap carefully.

- **Strategy quality varies wildly.** The core is well-audited; the 480+ community strategies are not uniformly maintained. Several popular OAuth/SAML strategies have had CVEs or gone unmaintained. Pin versions, check last-publish dates, and treat SAML strategies (`passport-saml` / `@node-saml/passport-saml`) with particular care — SAML signature-validation bugs have historically been the highest-severity issues in this ecosystem.

- **No CSRF, no rate limiting, no password hashing.** Passport authenticates; it does not protect the login route. You must add CSRF protection, brute-force throttling, and password hashing (bcrypt/argon2) yourself.

- **Serverless and edge are awkward.** The middleware + `express-session` model assumes a long-lived server and a session store. It works under serverless with an external store, but the ergonomics fight the platform; purpose-built auth libraries fit better there.

## When to Use / When Not

**Use when:**
- You are on Express (or Connect-compatible) and want a stable, unopinionated auth layer.
- You need a specific provider or protocol that already has a maintained strategy.
- You want full control over routes, session shape, and the user store rather than a framework's conventions.

**Avoid when:**
- You want batteries-included auth (UI, CSRF, MFA, account linking, email flows) out of the box.
- You are on Next.js, Fastify, Koa, or a serverless/edge runtime — the fit is poor and better-integrated options exist.
- You want an actively evolving core; Passport is stable-to-dormant and most innovation is happening elsewhere.

## Alternatives

- nextauthjs/next-auth (Auth.js) — use instead when you are on Next.js or want a provider-config-driven, session-managed solution with less wiring.
- lucia-auth/lucia — use instead when you want a lightweight, typed, framework-agnostic session/auth primitive you control (note: the library has been repositioned as a learning resource rather than a maintained dependency).
- better-auth/better-auth — use instead for a modern, TypeScript-first, batteries-included auth framework with plugins for MFA, orgs, and 2FA.
- panva/node-openid-client — use instead when you specifically need a spec-compliant OpenID Connect relying-party client rather than a generic strategy wrapper.
- Auth0 / Clerk / WorkOS / Stytch (hosted) — use instead when you would rather outsource authentication entirely and integrate via SDK.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2011–2012 | Initial releases; strategy pattern and Express middleware established[^1]. |
| 0.2.0 | 2014 | Serialize/deserialize and session handling stabilized. |
| 0.4.0 | 2017 | Long-lived stable line used across the Express 4 era. |
| 0.5.0 | 2021 | Maintenance release; groundwork for session-fixation fix. |
| 0.6.0 | 2022 | Session regeneration for fixation defense; `req.logout()` became async and callback-required[^4]. |
| 0.7.0 | 2024 | Latest core line; incremental fixes, Express 5 compatibility work[^3]. |

## References

[^1]: Passport README and project history — jaredhanson/passport. https://github.com/jaredhanson/passport
[^2]: passport.org strategy directory ("over 480 strategies"). https://www.passportjs.org/packages/
[^3]: GitHub API repository metadata for jaredhanson/passport — 23,500+ stars, MIT, last core push 2024-08-16, default branch `master` (fetched 2026-07). https://github.com/jaredhanson/passport
[^4]: Passport 0.6.0 release / logout change — `req.logout()` is now asynchronous and requires a callback. https://github.com/jaredhanson/passport/releases

## Tags

javascript, nodejs, authentication, express, middleware, oauth, openid-connect, saml, session-management, passport
