# paralleldrive/cuid2

> Collision-resistant string ids that are hashed, not sortable — deliberately slow enough to resist parallel guessing.

[GitHub repo](https://github.com/paralleldrive/cuid2) ·
npm: [@paralleldrive/cuid2](https://www.npmjs.com/package/@paralleldrive/cuid2) ·
[License: MIT](https://github.com/paralleldrive/cuid2/blob/main/LICENSE)

## Overview

Cuid2 is a small JavaScript library for generating unique string identifiers,
authored under Eric Elliott's Parallel Drive. It is the successor to the
original `cuid`, which is now deprecated in favor of this rewrite[^1]. The
package ships as `@paralleldrive/cuid2` and reports on the order of 10 million
weekly npm downloads, so despite the modest star count (~3.4k) it is a widely
depended-on primitive rather than an experiment.

The design goal is narrow and opinionated: produce ids that are hard to guess,
extremely unlikely to collide across uncoordinated hosts, and that leak nothing
about the data or the machine that made them. It does this by mixing several
independent entropy sources — an initial letter, the system time, a
pseudorandom value, a session counter, and a host fingerprint — and hashing the
combination through SHA3, then Base36-encoding the result to lowercase
`[a-z0-9]`[^1]. The default id is 24 characters, and length is configurable
down to a short slug.

The defining tension is that Cuid2 is intentionally *not too fast*. Its author
argues that an id generator fast enough to run in a render loop is also fast
enough to be brute-forced in parallel to find collisions or defeat
entropy-hiding, so the library trades raw throughput for a security property.
That is a real and unusual stance — most competitors optimize for speed — and
it is the single fact that should drive the choice to adopt or avoid it. Cuid2
is also, by design, **not** k-sortable/monotonic, which breaks a common habit of
using ids as a creation-time proxy.

## Getting Started

```bash
npm install @paralleldrive/cuid2
```

```js
import { createId, isCuid, init } from "@paralleldrive/cuid2";

createId();          // 'tz4a98xxat96iws9zmbrgj3a'  (24 chars, default)
isCuid(createId());  // true
isCuid("not a cuid") // false

// init() returns a custom generator; all options are optional.
const shortId = init({
  length: 10,                 // 50% collision odds after ~51M ids
  fingerprint: "my-host",     // extra host entropy for distributed setups
  random: Math.random,        // swap in a CSPRNG if you want
});
shortId(); // 'nw8zzfaa4v'
```

There is also a CLI (`npx @paralleldrive/cuid2`, with an optional `cuid` shell
alias) supporting `--slug`, `--length`, `--fingerprint`, and a `count`
argument.

## Architecture / How It Works

Each id concatenates: a leading letter (so the id is a valid JS/CSS/HTML
identifier), then the Base36 hash of `time + counter + random + fingerprint +
salt`. The salt and multiple entropy sources are the point — rather than
trusting one PRNG, Cuid2 combines several so a weakness in any one (for
example, a browser `Math.random()` that is known historically to have been
poor) does not sink the whole id[^1].

Key internal decisions, and why they matter:

- **SHA3 hashing over all inputs.** Because everything is hashed, no source
  (the timestamp, the host fingerprint) is recoverable from the output. This is
  the "leaks nothing" guarantee and also why ids are not sortable — the hash
  destroys any monotonic structure.
- **Host fingerprint from global names.** Cuid2 hashes the list of global
  identifiers present in the environment. This gives good per-host entropy in
  browsers, but in production Node the fingerprint can be near-identical across
  cloned containers/micro-VMs; the library compensates by seeding with random
  entropy so collision resistance survives even when the fingerprint is weak[^1].
- **Randomized session counter.** The counter is initialized to a random value
  and uses full JS number precision, so a single id extends the random entropy
  instead of wasting digits — an explicit fix for a wasteful pattern in v1.
- **Parameterized length via a shared hash.** Because the output is a hash, you
  can safely take any substring shorter than ~32 chars. Shorter means less
  entropy: the README gives the birthday-bound estimate
  `sqrt(36^(n-1) * 26)`, e.g. a 4-char slug reaches 50% collision odds after
  only ~1,100 ids, while the 24-char default needs ~4.03e18[^1].

The whole library is tiny (advertised under 5 KB gzipped) and fully
synchronous — no async entropy pool, no network round-trip.

## Production Notes

- **Do not use Cuid2 ids for sorting or as a creation-time proxy.** They are
  not k-sortable by design. If you need time ordering, add an indexed
  `createdAt` column; do not reach for ULID/UUIDv7 behavior here[^1].
- **Database index locality.** Random (non-monotonic) primary keys fragment
  B-tree indexes and hurt insert locality on large tables — the same cost that
  motivated UUIDv7/ULID. On write-heavy Postgres/MySQL this is a real
  consideration; many teams keep a monotonic internal PK and expose the cuid
  externally.
- **The "not too fast" property is a feature, not a bug.** Do not swap the
  hash or call it in a hot render loop expecting nanoid-class throughput. If you
  need raw speed and don't need the security/anti-collision guarantees, the
  README itself points you to a plain counter, Ulid, or NanoId.
- **Fingerprint weakness in identical containers.** In homogeneous
  cloud/serverless fleets the host fingerprint contributes little; rely on the
  random entropy and consider passing a distinct `fingerprint` per host/worker
  via `init()` if you generate ids at very high concurrency.
- **Length is a security knob.** Shortening ids for prettier URLs measurably
  lowers collision resistance and guessability margins. Keep the 24-char
  default for anything database- or security-adjacent.
- **Validation is shallow.** `isCuid()` checks shape/format, not authenticity —
  it cannot tell you an id was actually produced by your generator.

## When to Use / When Not

**Use when:**
- You generate ids on many uncoordinated clients/servers (offline-first, edge,
  multi-writer) and want collision resistance without a central sequence.
- Ids are exposed to users and must not be guessable or leak timing/host info.
- You want a small, synchronous, dependency-light generator with a stable API.

**Avoid when:**
- You need time-sortable ids for index locality or range scans — use UUIDv7 or
  ULID instead.
- You are in a tight/high-throughput loop where per-id latency matters; the
  deliberate slowness works against you.
- You need a fixed binary/128-bit format for interop with systems that expect
  UUIDs specifically.

## Alternatives

- ai/nanoid — smaller and faster, URL-safe random ids; use when you want speed
  and a custom alphabet and don't need the multi-source/hardened entropy story.
- ulid/javascript — lexicographically sortable, timestamp-prefixed ids; use
  when you need creation-time ordering that Cuid2 deliberately refuses.
- uuidjs/uuid — the standard UUID formats (incl. v4 random, v7 time-ordered);
  use when a system on the other end requires canonical UUIDs.
- segmentio/ksuid — k-sortable 20-byte ids with an embedded timestamp; use for
  sortable event/log ids at scale.
- jetify-com/typeid — type-prefixed, UUIDv7-based ids; use when you want
  human-readable type tags plus sortability.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2022-12-26 | `paralleldrive/cuid2` first published; rewrite of the original `cuid`[^2]. |
| 1.x | 2023 | Initial stable Cuid2 line; original `cuid` marked deprecated[^1]. |
| 2.x | — | Current major line on npm; adds CLI (`--install` alias, `--slug`, `--length`). |
| — | 2026-06-09 | Latest push to `main` as of this writing[^2]. |

## References

[^1]: Cuid2 README, paralleldrive/cuid2 — design rationale, entropy sources,
SHA3 hashing, collision math, and "not too fast" argument.
https://github.com/paralleldrive/cuid2#readme
[^2]: GitHub REST API, `repos/paralleldrive/cuid2` — created_at 2022-12-26,
pushed_at 2026-06-09, MIT, ~3.4k stars, 72 forks (fetched 2026-07-15).
https://github.com/paralleldrive/cuid2

## Tags

javascript, typescript, unique-id, uuid-alternative, nanoid-alternative, collision-resistant, distributed-systems, cryptography, sha3, identifiers, npm-package
