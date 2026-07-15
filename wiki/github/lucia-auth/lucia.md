# lucia-auth/lucia

> An auth library that deprecated itself — now a learning resource for implementing sessions from scratch in TypeScript.

[GitHub repo](https://github.com/lucia-auth/lucia) ·
[Official website](https://lucia-auth.com) ·
[License: 0BSD](https://github.com/lucia-auth/lucia/blob/main/LICENSE-0BSD)

## Overview

Lucia began as a session-based authentication library for TypeScript, created by
the developer known as pilcrow (pilcrowOnPaper) and first released in 2023. Its
pitch was a database-agnostic, framework-agnostic session layer: you brought an
adapter for your ORM/driver, and Lucia handled session creation, validation, and
expiry without the redirect-heavy, provider-coupled model of NextAuth. It reached
v3 and a five-figure star count on that premise.

In 2024 the maintainer reversed the entire thesis. The announcement (discussion
#1714) stated that Lucia v3 would be **deprecated by March 2025**, and the project
was reframed from a library into a *learning resource*: documentation that teaches
you to implement sessions directly against your own database[^1]. The current
repository is primarily the source for the lucia-auth.com guide, not a shippable
package. This is the defining tension of the project — it is one of the rare
popular repos whose author concluded the abstraction was not worth maintaining and
told users to write the ~100 lines themselves.

The reasoning was concrete: supporting every combination of database library, ORM,
framework, runtime, and deployment target, while staying flexible and not adding
project complexity, proved intractable. Sessions are simple enough that teaching
the code beats packaging it[^2]. What survives is not a dependency but a pattern,
plus two lower-level libraries (Oslo, Arctic) the same author extracted from it.

## Getting Started

There is no current library to install — the "getting started" is reading the guide
and copying code into your project. The canonical session pattern looks like this:

```ts
// session.ts — validate a session token against your own database
import { sha256 } from "@oslojs/crypto/sha2";
import { encodeHexLowerCase } from "@oslojs/encoding";

export async function validateSessionToken(token: string) {
  const sessionId = encodeHexLowerCase(sha256(new TextEncoder().encode(token)));
  const row = await db.getSession(sessionId); // your query
  if (!row) return null;
  if (Date.now() >= row.expiresAt.getTime()) {
    await db.deleteSession(sessionId);
    return null;
  }
  // sliding expiration: extend when close to expiry
  if (Date.now() >= row.expiresAt.getTime() - 1000 * 60 * 60 * 24 * 15) {
    row.expiresAt = new Date(Date.now() + 1000 * 60 * 60 * 24 * 30);
    await db.updateSessionExpiry(sessionId, row.expiresAt);
  }
  return row;
}
```

The token is a random secret; only its SHA-256 hash is stored, so a database leak
does not expose usable session tokens. You own the `db.*` calls entirely.

## Architecture / How It Works

The Lucia model is deliberately thin. A session is a row: an id (the hash of the
token), a user id, and an expiry timestamp. Authentication is two operations —
create a session (generate a random token, store its hash) and validate a session
(hash the incoming token, look up the row, check expiry, optionally extend). There
is no session store abstraction, no adapter interface, no plugin system in the
current guidance; those were exactly the parts the maintainer decided not to keep.

The old library (v3, preserved on the `v3` branch) worked differently: a `Lucia`
class configured with a database adapter and a session-cookie controller, exposing
`createSession`, `validateSession`, and cookie helpers. Adapters existed for
Drizzle, Prisma, better-sqlite3, postgres, mysql2, Mongoose, and others. That
adapter surface is the maintenance burden that motivated the deprecation.

The functionality did not disappear — it was factored into two runtime-agnostic
libraries the guide now leans on:

- **Oslo** (oslojs.dev) — small, fully typed packages for the cryptographic and
  encoding primitives auth needs: SHA-256, HMAC, random token generation, hex/base64
  encoding, JWT, cookies. Minimal dependencies, works across Node, Deno, Bun, and
  workers[^3].
- **Arctic** (arcticjs.dev) — an OAuth 2.0 client covering 50+ providers, for the
  social-login half that sessions alone do not address[^4].

The intellectual companion is **The Copenhagen Book**, a separate free resource by
the same author documenting auth concepts (session management, password hashing,
email verification, CSRF, MFA) independent of any library[^5].

## Production Notes

**The library is end-of-life — do not add it to new projects.** The v3 package is
deprecated and unmaintained as of 2025. If you find tutorials or starter templates
that `npm install lucia`, they are pointing at a frozen version that will not
receive security or compatibility fixes. Treat the repo as documentation.

**Migrating off v3.** Existing users cannot rely on a drop-in replacement. The
official path is to copy the session logic into your codebase (the guide provides
per-framework and per-database walkthroughs) and depend on Oslo/Arctic for
primitives. Practically this means moving auth from a dependency you upgrade to
code you own and test — more control, more responsibility. Budget real time for it;
this is a code migration, not a version bump.

**What you inherit by owning the code.** Session fixation, secure cookie flags
(`HttpOnly`, `Secure`, `SameSite`), token entropy, sliding-window expiration, and
CSRF are now yours to get right. The guide covers them, but there is no library
default protecting you. This is the correct model for teams who want to understand
their auth; it is a footgun for teams who wanted "auth handled."

**Licensing subtlety.** GitHub reports the repo as 0BSD, but the repository actually
carries two licenses: example/site code is Zero-Clause BSD (use, copy, modify, no
attribution — OSI-approved), while everything else is MIT[^6]. Copying snippets from
the guide into a commercial product is explicitly fine.

## When to Use / When Not

**Use (as a resource) when:**
- You want to understand session auth well enough to implement and own it.
- You need a vetted reference pattern for token hashing and expiry in TS.
- You are choosing primitives (Oslo, Arctic) rather than a batteries-included stack.

**Avoid when:**
- You want a maintained, installable auth library with community support — Lucia is
  no longer that.
- You need managed features out of the box: social login flows, MFA, admin UI,
  organizations, RBAC.
- Your team lacks the time or appetite to own and security-review auth code.

## Alternatives

- better-auth/better-auth — framework-agnostic TS auth library; the most-cited
  successor for teams who still want a library rather than a copy-paste guide.
- nextauthjs/next-auth (Auth.js) — provider-oriented session/OAuth for Next.js and
  other frameworks; use when social login and a plugin ecosystem matter more than
  owning the code.
- openauthjs/openauth — self-hosted OAuth/OIDC server; use when you want a standard
  auth server rather than embedded sessions.
- ory/kratos — full identity server (self-hosted API) for when auth is
  infrastructure, not a library concern.
- supabase/auth — managed auth service; use when you want a hosted backend and are
  already in that ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 | 2023 | First stable release as a session auth library. |
| v2 | 2023 | API redesign; adapter model refined. |
| v3 | 2024 | Latest and final library version; preserved on the `v3` branch. |
| Deprecation announced | 2024 | Discussion #1714: pivot to learning resource[^1]. |
| v3 deprecated | 2025-03 | Library end-of-life; repo becomes documentation[^1]. |

## References

[^1]: Lucia deprecation announcement, GitHub Discussion #1714. https://github.com/lucia-auth/lucia/discussions/1714
[^2]: "Why not a library?" — Lucia README. https://github.com/lucia-auth/lucia
[^3]: Oslo — auth and cryptography primitives. https://oslojs.dev
[^4]: Arctic — OAuth 2.0 client library. https://arcticjs.dev
[^5]: The Copenhagen Book — web auth concepts. https://thecopenhagenbook.com
[^6]: Lucia licensing (0BSD example code + MIT). https://github.com/lucia-auth/lucia/blob/main/LICENSE-0BSD
[^7]: Repository metadata (stars, forks, license, last push) via GitHub API, retrieved 2026-07-15.

## Tags

typescript, authentication, sessions, oauth, deprecated, learning-resource, security, web, javascript, self-hosted
