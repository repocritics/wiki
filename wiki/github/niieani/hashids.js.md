# niieani/hashids.js

> Reversibly turn one or more non-negative integers into a short, URL-safe, YouTube-style string — id obfuscation, explicitly not encryption.

[GitHub repo](https://github.com/niieani/hashids.js) ·
[Official website](https://hashids.org/javascript) ·
[License: MIT](https://github.com/niieani/hashids.js/blob/master/LICENSE)

## Overview

Hashids is a tiny (~5 KB), zero-dependency library that maps arrays of
non-negative integers to short reversible strings and back:
`encode(1, 2, 3) → "o2fXhV"`, `decode("o2fXhV") → [1, 2, 3]`. The typical use
is hiding sequential database ids in URLs so `/user/1` becomes `/user/o2fXhV`,
avoiding the enumeration and "how many customers do you have" leakage that raw
auto-increment ids create. `niieani/hashids.js` is the canonical JavaScript /
TypeScript port, maintained by Bazyli Brzóska; the algorithm and the
`hashids.org` brand originate with Ivan Akimov, who wrote the first reference
implementations in 2012[^1].

The name is the project's defining tension. Hashids is neither a hash (hashes
are one-way; this is fully reversible) nor a cipher. The "salt" is not a key —
it only permutes the alphabet — and the whole algorithm is public, so anyone
holding a few outputs can, with effort, recover the salt and decode everything.
The README is blunt about this: "Do not use this library as a security tool and
do not encode sensitive data. This is not an encryption library."[^2] Treat the
output as obfuscation that raises the cost of casual enumeration, nothing more.

The other thing to know up front: in 2023 the original author released
**Sqids** (`sqids.org`) as the explicit successor to Hashids, citing the
misleading name and design cleanups, and `hashids.org` now points visitors
there[^3]. `niieani/hashids.js` still works and is widely depended on, but it is
in quiet maintenance — the last published release, 2.3.0, is from May 2023[^4].

## Getting Started

```shell
npm install hashids      # or: yarn add hashids
```

```javascript
import Hashids from 'hashids'          // ESM
// const Hashids = require('hashids/cjs')  // CommonJS

const hashids = new Hashids('my project salt', 8)  // salt, min length

const id = hashids.encode(1, 2, 3)     // "El3fkRTaA0" — at least 8 chars
const nums = hashids.decode(id)        // [1, 2, 3]  (always an array)

// hex helper, e.g. for Mongo ObjectIds — preserves leading zeros:
hashids.encodeHex('507f1f77bcf86cd799439011')  // "y42LW46J9luq3Xq9XMly"
```

Constructor signature: `new Hashids(salt = '', minLength = 0, alphabet = default,
seps)`. The same four arguments must be reused to decode — a different salt,
minimum length, or alphabet yields different (and non-reversible) output.

## Architecture / How It Works

Encoding is a deterministic permutation, not a lookup table:

1. A **lottery** character is chosen from the alphabet using the sum of the
   input numbers, then prepended. This is why `encode(5, 5, 5)` and
   `encode(1, 2, 3, ...)` do not produce visually repeating or incrementing
   output — each number is encoded against an alphabet that was reshuffled using
   the lottery char plus the salt.
2. Each number is base-converted into the (consistently shuffled) alphabet.
3. **Separator** characters (drawn from `cfhistuCFHISTU`) are interleaved between
   encoded numbers so `decode` can split them back apart.
4. If `minLength` is set, **guard** characters and alphabet padding are added on
   both ends until the string is at least that long. Padding is "at least" —
   ids can exceed the requested length, never guaranteed to equal it.

Decoding reverses these steps using the same salt-derived shuffle. `encodeHex`
splits a hex string into chunks, prefixes each with `1` to preserve leading
zeros, and encodes them as ordinary integers — which is why it round-trips
length information that a plain `BigInt` encode of the same value would lose.

Two aesthetic behaviors are baked in. By default the algorithm avoids placing
the letters `c f h i s t u` adjacent to each other, a crude filter against
common English profanity appearing in URLs; the offending set is configurable
via the 4th constructor argument. Since 2.0 the alphabet may contain arbitrary
Unicode, including emoji. The custom alphabet must contain a minimum number of
unique characters (16) or the constructor throws.

## Production Notes

- **The salt is not a secret and not a key.** Same salt in, same output out; the
  mapping is stable and public. Do not treat hashids as access control — always
  authorize the decoded id server-side as if it were the raw integer, because an
  attacker who guesses or brute-forces neighboring ids gets valid ids back.
- **Config is load-bearing forever.** Salt, `minLength`, and `alphabet` are part
  of your data contract. Changing any of them after ids are in the wild
  (emails, bookmarks, external references) breaks every previously issued id.
  Pin them and treat changes as a migration.
- **Non-negative integers only.** Negative numbers are unsupported; floats are
  not meaningful. Large values beyond `Number.MAX_SAFE_INTEGER` need the BigInt
  API (`encode(1n)`), which throws on runtimes without `BigInt`.
- **Silent empty string on bad input.** `encode('123a')` returns `''` rather
  than throwing. Validate inputs yourself if a caller might pass junk, or you
  will store/emit empty ids.
- **Not collision-proof across different inputs by construction of your keys** —
  hashids is a bijection for a *fixed* config, but two different salts/alphabets
  can produce the same string for different numbers. Never mix configs in one
  id space.
- **Module formats.** v13-era Node conditional exports are advertised, but in
  practice you import `hashids` (ESM) or `hashids/cjs` (CommonJS); TypeScript
  ESM users need `esModuleInterop`/`allowSyntheticDefaultImports`, and BigInt
  typing may require adding `esnext.bigint` to `tsconfig` `lib`.
- **Maintenance cadence.** No releases since 2.3.0 (May 2023). For existing
  systems this is fine — the algorithm is frozen and the code is stable. For new
  systems, weigh Sqids first (below).

## When to Use / When Not

**Use when:**
- You want to hide sequential/internal integer ids in URLs or public payloads to
  discourage casual enumeration and scraping.
- You need a short, human-copyable, reversible token derived from numbers with
  no storage lookup.
- You are maintaining an existing system already issuing hashids and need
  compatibility.

**Avoid when:**
- You need actual confidentiality or integrity — use encryption (AES-GCM) or
  signed tokens (HMAC / JWT); hashids provides neither.
- You are starting fresh: prefer Sqids, the maintained successor from the same
  author, unless you specifically need drop-in hashids compatibility.
- You need globally unique ids without a source integer (use nanoid/UUID), or
  sortable time-based ids (use ULID).

## Alternatives

- sqids/sqids-javascript — the official successor by the original author; use for
  any new project unless you need bug-for-bug hashids compatibility.
- ai/nanoid — use when you want short random unique ids rather than reversible
  obfuscation of an existing counter.
- uuidjs/uuid — use when you need standardized collision-resistant unique ids and
  don't care about length or reversibility.
- ulid/javascript — use when you want lexicographically sortable, timestamp-
  embedded ids.
- Application-level encryption (AES-GCM) or signed tokens — use when the value
  must actually be secret or tamper-evident, which hashids never provides.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2014-09-09 | Early stable JS implementation of the Hashids algorithm[^5]. |
| 1.2.1 | 2018-09-16 | Last of the 1.x line. |
| 2.0.0 | 2019-08-31 | TypeScript rewrite; BigInt support; arbitrary/emoji alphabets[^6]. |
| 2.2.0 | 2020-02-03 | Maintenance and packaging improvements. |
| 2.2.11 | 2023-02-16 | Bug-fix maintenance release. |
| 2.3.0 | 2023-05-24 | Latest release; library effectively feature-complete[^4]. |

## References

[^1]: Repository created 2012-08-11; project brand and reference implementations
by Ivan Akimov. https://hashids.org/
[^2]: hashids.js README, "Pitfalls" and "Randomness" sections.
https://github.com/niieani/hashids.js#pitfalls
[^3]: Sqids — "Hashids is now Sqids." https://sqids.org/
[^4]: Release 2.3.0, 2023-05-24 (most recent published release, per GitHub
Releases as of 2026-07). https://github.com/niieani/hashids.js/releases
[^5]: Tag 1.0.0 commit, 2014-09-09.
https://github.com/niieani/hashids.js/releases/tag/1.0.0
[^6]: Release 2.0.0, 2019-08-31.
https://github.com/niieani/hashids.js/releases/tag/2.0.0

## Tags

javascript, typescript, id-obfuscation, encoding, url-ids, database-ids, hashids, sqids, npm-package, zero-dependency
