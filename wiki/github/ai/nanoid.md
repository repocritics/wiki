# ai/nanoid

> A ~118-byte, dependency-free unique string ID generator that trades UUID's standard format for shorter, URL-safe IDs.

[GitHub repo](https://github.com/ai/nanoid) ·
[ID collision calculator](https://zelark.github.io/nano-id-cc/) ·
[License: MIT](https://github.com/ai/nanoid/blob/main/LICENSE)

## Overview

Nano ID is a small JavaScript library that generates random, URL-friendly identifiers. It was created by Andrey Sitnik (author of PostCSS and Autoprefixer) at Evil Martians and first released in 2017[^1]. The default output is a 21-character string drawn from a 64-symbol alphabet (`A-Za-z0-9_-`), carrying ~126 bits of entropy — comparable to a random UUID v4's 122 bits, but in 21 characters instead of 36.

The project's defining stance is minimalism as a discipline: the README's own tagline calls it "an amazing level of senseless perfectionism." The secure entry point is a handful of lines that pull bytes from a hardware CSPRNG (`crypto` in Node, Web Crypto in browsers) and map them onto an alphabet using rejection sampling rather than the naive `random % alphabet` that biases distribution. That is essentially the entire product — there is no configuration surface beyond alphabet, size, and an injectable RNG.

The central tradeoff is format, not quality. Nano ID's IDs are not a recognized standard, so any system that expects RFC 4122 UUIDs (Postgres `uuid` columns, some ORMs, external APIs) will treat them as opaque text rather than a native UUID type. In exchange you get shorter IDs, a smaller bundle, and a customizable alphabet. With ~27k stars and steady commits into 2026[^2], it is one of the most widely depended-on ID libraries in the npm ecosystem, frequently pulled in transitively rather than chosen directly.

## Getting Started

```bash
npm install nanoid
```

```js
import { nanoid } from 'nanoid'

model.id = nanoid()      //=> "V1StGXR8_Z5jdHi6B-myT"  (21 chars)
model.id = nanoid(10)    //=> "IRFa-VaY2b"             (shorter, more collisions)
```

```js
// Custom alphabet + fixed size — returns a configured generator
import { customAlphabet } from 'nanoid'

const numericId = customAlphabet('0123456789', 6)
numericId()              //=> "480274"
```

## Architecture / How It Works

The library is a few tiny modules with no runtime dependencies. Three properties drive the whole design:

- **CSPRNG bytes, not `Math.random()`.** The secure build fills a `Uint8Array` from `crypto.getRandomValues` / Node's `crypto`. This is what makes IDs safe to use as tokens and safe across clustered processes.
- **Rejection sampling for uniformity.** A byte is 0–255; mapping it onto, say, a 62-symbol alphabet with modulo would over-represent low symbols. Nano ID instead masks each byte to the smallest power-of-two window covering the alphabet and discards out-of-range draws, so every symbol is equiprobable. This is the "better algorithm" the README references and the reason `customAlphabet` requests random bytes in a loop rather than one shot.
- **Alphabet ≤ 256 symbols.** Because the generator works a byte at a time, alphabets larger than 256 characters break the uniformity guarantee; the docs state this as a hard constraint.

There are three distinct entry points that matter in practice: `nanoid` (secure default), `nanoid/non-secure` (a `Math.random()`-based variant that is faster to *start* but weaker), and `customAlphabet` / `customRandom` for configured generators. The non-secure build exists mainly for environments without a hardware RNG; the README explicitly notes it is *slower* per-call than the secure version in Node, so it is not a performance optimization.

Nano ID is deliberately not sortable and carries no embedded timestamp — IDs are pure random noise. Systems that want lexicographic or time ordering (for database index locality) need ULID/KSUID instead; this is a design boundary, not a missing feature.

## Production Notes

- **ESM-only from v4.** Nano ID 4.0 (2022) dropped CommonJS and shipped as pure ESM, and v5 continues this[^3]. In a CommonJS project (`require('nanoid')`) this surfaces as `ERR_REQUIRE_ESM`. The pragmatic escapes are: pin to the `3.x` line (still maintained, dual CJS/ESM), use dynamic `import()`, or move the consumer to ESM. This single change is the most common upgrade complaint and the reason `nanoid@3` still sees enormous download volume.
- **Security advisory CVE-2024-55565.** A non-integer `size` argument (e.g. a float from unvalidated input) could cause incorrect behavior; it was fixed in 3.3.8 and 5.0.9[^4]. If you pass a user-supplied length, validate it as a positive integer and stay on a patched release.
- **Collision math is your responsibility.** Shortening IDs with `nanoid(n)` cuts entropy fast. At the 21-char default, collisions are effectively impossible at realistic volumes, but a 6–10 char ID in a high-throughput system will collide. Use the linked collision calculator before shrinking, and prefer a database unique constraint as a backstop.
- **React `key` misuse.** A recurring footgun the README calls out: never use `nanoid()` for a React list `key`, because it produces a new value every render and defeats reconciliation. Use a stable record ID or, for accessibility linking, React's `useId`.
- **React Native / restricted runtimes.** RN has no built-in secure RNG; you must install `react-native-get-random-values` and import it before Nano ID. Some serverless or embedded JS runtimes without Web Crypto need a similar polyfill or the non-secure build.
- **PouchDB/CouchDB.** IDs may begin with `_`, which those databases reserve. Prefix the ID (`'id' + nanoid()`) to avoid rejected documents.

## When to Use / When Not

**Use when:**
- You want short, URL-safe, high-entropy IDs and don't need the UUID format.
- Bundle size matters (client-side ID generation, edge functions).
- You need a custom alphabet — numeric-only, no-lookalikes, profanity-filtered — via `customAlphabet`.

**Avoid when:**
- A system expects real RFC 4122 UUIDs (native DB `uuid` columns, strict external APIs) — use a UUID library instead.
- You need time-sortable or lexicographically ordered IDs for index locality — reach for ULID/KSUID.
- You're on Node/modern browsers and want zero dependencies with a standard format — the built-in `crypto.randomUUID()` may be enough.

## Alternatives

- uuidjs/uuid — the ecosystem-standard RFC 4122 implementation; use when you need canonical UUID format and interop.
- lukeed/uid — even smaller, non-cryptographic; use when you only need cheap unique-ish strings and not security.
- paralleldrive/cuid2 — collision-resistant, hardened against fingerprinting; use when you want stronger guarantees than raw randomness.
- ulid/javascript — 128-bit, lexicographically sortable, timestamp-prefixed; use when ID ordering aids DB indexing.
- Node/Web `crypto.randomUUID()` — built-in, zero-dependency; use when a standard UUID and no npm install is preferable.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-01 | First stable release; secure + non-secure generators. |
| 2.0 | 2019 | API cleanup, custom alphabet helpers. |
| 3.0 | 2020-02 | Rewrite; faster, `customAlphabet`/`customRandom`, dual CJS/ESM. Long-lived LTS-style line. |
| 4.0 | 2022-01 | Pure ESM; CommonJS support dropped[^3]. |
| 5.0 | 2023-10 | Current major; ESM, Web Crypto in browsers, JSR distribution. |

## References

[^1]: `ai/nanoid` repository, created 2017-08-05. https://github.com/ai/nanoid
[^2]: GitHub API repo metadata (stars, last push) fetched 2026-07-15. https://api.github.com/repos/ai/nanoid
[^3]: Nano ID CHANGELOG — 4.0 ESM-only migration. https://github.com/ai/nanoid/blob/main/CHANGELOG.md
[^4]: Nano ID security advisories (CVE-2024-55565, fixed in 3.3.8 / 5.0.9). https://github.com/ai/nanoid/security/advisories

## Tags

javascript, id-generator, uuid-alternative, url-safe, csprng, security, esm, tiny-library, npm, unique-id, browser
