# unjs/ohash

> Deterministic object hashing, stable serialization, and diffing for JavaScript — the fingerprinting primitive behind much of the UnJS/Nuxt cache layer.

[GitHub repo](https://github.com/unjs/ohash) ·
[npm](https://www.npmjs.com/package/ohash) ·
[License: MIT](https://github.com/unjs/ohash/blob/main/LICENSE)

## Overview

ohash is a small utility from the UnJS collective that turns any JavaScript
value into a stable string — either a serialized form or a fixed-length hash —
so that two structurally equal values produce the same output regardless of key
order or reference identity. It is the kind of dependency you never install
directly but end up shipping anyway: it underpins cache keys, payload
deduplication, and change detection across Nuxt, Nitro, and other UnJS
tooling[^1]. Its serialization logic descends from Scott Puleo's
`object-hash`, and its bundled SHA-256 from `crypto-js`[^2].

The defining tension is stability versus security. ohash exists to answer "are
these two things the same?" quickly and portably (browser, Node, Deno, edge),
not "can an adversary forge a collision?" The maintainers are explicit that
`serialize` makes a best effort at stable output but is **not** designed for
security, and that intentional collisions from untrusted input are always
possible[^3]. Treating an ohash digest as a tamper-evident signature is a
misuse, not a bug.

The other thing to know up front: **v2 is a rewrite with a different API** from
the widely-deployed v1, and the two are not drop-in compatible. Code and blog
posts written against v1 (`objectHash`, `murmurHash`, `sha256base64`,
options-heavy `hash(obj, options)`) do not describe the current package[^4].

## Getting Started

```sh
# auto-detects npm / yarn / pnpm / bun / deno
npx nypm i ohash
```

```js
import { hash, serialize, digest, isEqual } from "ohash";
import { diff } from "ohash/utils";

hash({ foo: "bar" });
// "g82Nh7Lh3CURFX9zCBhc5xgU0K7L0V1qkoHyRsKNqA4"

// order-independent: same input, same hash
hash({ a: 1, b: 2 }) === hash({ b: 2, a: 1 }); // true

serialize({ foo: "bar" });          // "{foo:'bar'}"
digest("Hello World!");             // "f4OxZX_x_FO5LcGBSKHWXfwtSx-j1ncoSt3SABJtkGk"
isEqual({ a: 1, b: 2 }, { b: 2, a: 1 }); // true
```

## Architecture / How It Works

`hash(input)` is a two-stage pipeline, and understanding the split is the key to
using ohash well:

1. **`serialize(input)`** walks the value and emits a compact, deterministic
   string representation (`{foo:'bar'}`). Object keys are ordered so that
   `{a,b}` and `{b,a}` serialize identically. This stage handles the JS-specific
   awkward cases — nested objects, arrays, and non-JSON types — that a plain
   `JSON.stringify` would get wrong or refuse.
2. **`digest(str)`** runs SHA-256 over the serialized string and encodes the
   result as Base64URL, yielding the fixed-length token you see from `hash`.

Because the two stages are exposed separately, you can serialize once and reuse
the string, or feed your own hash function, or hash a value that was serialized
elsewhere. `isEqual(a, b)` short-circuits on `===` and otherwise compares
serialized forms, so it is a value-equality check that costs a serialization
pass, not a hash.

`diff(a, b)` (from `ohash/utils`) is the heavier tool: it serializes nested
structures and returns an array of change entries — each with `$key`, `$hash`,
`$value`, and `$props` — describing what was added, removed, or changed between
two objects. It stringifies to a readable changelog (`[+] Added`, `[-] Removed`,
`[~] Changed`), which makes it useful for cache-invalidation logging and config
drift detection.

The SHA-256 implementation is bundled rather than delegated to
`crypto.subtle`/Node `crypto`, which is what makes ohash run unchanged in the
browser, edge runtimes, and workers without an async crypto API or a Node
polyfill. That portability is deliberate and is a large part of why the UnJS
stack depends on it.

## Production Notes

- **Not a security primitive.** The digest is SHA-256, but the *serialization*
  in front of it is not collision-resistant against a motivated attacker who
  controls the input. Do not use ohash to sign, authenticate, or dedupe
  security-sensitive data. For that, serialize deterministically yourself and
  HMAC with a real crypto library[^3].
- **Hash strings are not stable across major versions.** The serialization
  format is an implementation detail. If you persist ohash outputs (as cache
  keys in Redis, on disk, in a CDN), a ohash upgrade — especially v1→v2 — can
  change the string for identical inputs and silently invalidate or, worse,
  mismatch your entire cache. Pin the version and treat a bump as a cache flush.
- **v1 → v2 is a breaking API change.** v1's named exports (`objectHash`,
  `murmurHash`, `sha256`, `sha256base64`) and its options object are gone or
  reshaped in v2. The v2 surface is the small `{ hash, serialize, digest,
  isEqual }` core plus `diff` under `ohash/utils`. Migrate against the v2 release
  notes, not v1 documentation[^4].
- **Functions, symbols, and exotic values.** ohash serializes many
  non-JSON types, but the representation of things like functions, class
  instances, `Date`, `Map`/`Set`, and circular references is defined by ohash,
  not by any standard. Verify the behavior for your specific types before
  relying on it — two values you consider "equal" may serialize differently, or
  vice versa.
- **Cost scales with the value.** `hash`/`isEqual`/`diff` all serialize the full
  structure every call; there is no structural sharing or memoization. Hashing
  large payloads in a hot path is a real CPU cost — cache the digest, don't
  recompute it per request.

## When to Use / When Not

**Use when:**
- You need a stable cache key or dedup key from an arbitrary JS object.
- You want order-independent structural equality without deep-equal libraries.
- You need the same hashing behavior in the browser, Node, Deno, and edge.
- You want a readable diff of two config/state objects.

**Avoid when:**
- You need cryptographic integrity, signatures, or tamper detection.
- You persist hashes long-term across dependency upgrades without a flush plan.
- You only need deterministic JSON and can hash it yourself (skip the dependency).
- You are hashing very large objects in a latency-critical loop without caching.

## Alternatives

- puleos/object-hash — the Node-focused ancestor ohash's serialization derives
  from; use it when you want the mature, options-rich original and don't need
  edge/browser portability.
- brix/crypto-js — use when you need actual cryptographic hashing/HMAC over
  strings, not object fingerprinting.
- planttheidea/hash-it — use for a tiny, fast integer object hash when a
  Base64URL string and diffing aren't needed.
- epoberezkin/fast-json-stable-stringify — use when you only need deterministic
  serialization and will feed the string to your own hash.
- unjs/ufo, unjs/destr — unrelated, but the same UnJS reliability/portability
  design philosophy if you're evaluating the ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-10-18 | Repository created; v1 line based on object-hash + crypto-js[^2]. |
| 1.x | 2022–2024 | Widely adopted across UnJS/Nuxt; `objectHash`/`murmurHash`/`sha256*` API. |
| 2.0 | 2025 | Rewrite: `{ hash, serialize, digest, isEqual }` core + `ohash/utils` `diff`; migration notes at v2.0.1[^4]. |

## References

[^1]: UnJS project index — ohash. https://unjs.io/packages/ohash
[^2]: ohash README — attribution to puleos/object-hash and brix/crypto-js. https://github.com/unjs/ohash#license
[^3]: ohash README — `serialize` "is not designed for security purposes … there is always a chance of intentional collisions." https://github.com/unjs/ohash#serializeinput
[^4]: ohash README v2 note and v2.0.1 release/migration notes. https://github.com/unjs/ohash/releases/tag/v2.0.1

## Tags

typescript, javascript, hashing, serialization, object-hash, sha256, caching, diffing, unjs, deterministic
