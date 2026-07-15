# servo/rust-url

> WHATWG-conformant URL parser for Rust — the `url` crate the ecosystem parses on.

[GitHub repo](https://github.com/servo/rust-url) ·
[Documentation](https://docs.rs/url/) ·
[License: MIT OR Apache-2.0](https://github.com/servo/rust-url/blob/main/LICENSE-APACHE)

## Overview

rust-url is a Cargo workspace that produces `url` and a handful of lower-level
crates (`idna`, `percent-encoding`, `form_urlencoded`, `data-url`, `idna_adapter`).
The headline crate, `url`, implements the [WHATWG URL Standard](https://url.spec.whatwg.org/)[^1] —
the same living specification browsers follow — rather than RFC 3986. It began
inside Mozilla's Servo browser engine (primary author Simon Sapin) as the URL
parser a real browser needs, and is now one of the most-depended-upon crates on
crates.io: `reqwest`, `hyper`-adjacent tooling, `cargo`, and most HTTP clients
pull it in transitively.

The defining tradeoff is **WHATWG semantics, not RFC semantics**. `Url::parse`
does not merely validate — it *normalizes*. It lowercases the scheme and host,
resolves `.`/`..` path segments, applies IDNA to non-ASCII hosts, and forces a
`/` path onto "special" schemes (`http`, `https`, `ws`, `wss`, `ftp`, `file`).
The parse result is therefore a canonicalized URL, which is exactly what you want
for fetching and comparison and exactly what surprises anyone expecting a
byte-preserving RFC 3986 parser. If you need a URL to round-trip unchanged, this
is the wrong crate.

The second defining trait is that internationalized domain handling (`idna`) is
heavy. By default `idna` pulls its Unicode tables from ICU4X, which raises the
minimum supported Rust version and binary size compared to a naïve ASCII-only
parser — a cost you inherit transitively even for URLs that are pure ASCII.[^2]

## Getting Started

```bash
cargo add url
# optional: cargo add url --features serde
```

```rust
use url::Url;

fn main() -> Result<(), url::ParseError> {
    let url = Url::parse("https://user@Example.COM:443/a/../b?q=1#frag")?;

    assert_eq!(url.scheme(), "https");
    assert_eq!(url.host_str(), Some("example.com")); // host lowercased
    assert_eq!(url.path(), "/b");                    // dot-segment resolved
    assert_eq!(url.port_or_known_default(), Some(443));
    assert_eq!(url.query(), Some("q=1"));
    assert_eq!(url.fragment(), Some("frag"));

    // Relative resolution against a base
    let joined = url.join("../c?x=2")?;
    assert_eq!(joined.as_str(), "https://user@example.com/c?x=2");
    Ok(())
}
```

## Architecture / How It Works

The `Url` type is not a struct of owned component strings. It stores the full
serialized URL as a single `String` plus a set of `u32` offsets marking where the
scheme, username, host, port, path, query, and fragment begin and end.[^3]
Accessors like `.host_str()` or `.path()` return `&str` slices into that one
buffer. Consequences of this design:

- A `Url` is always in canonical, valid serialized form — there is no way to
  construct an internally inconsistent one through the safe API.
- Reading components is allocation-free; the serialization (`as_str()`) is free.
- Mutation is comparatively awkward: setters (`set_host`, `set_path`,
  `set_query`) must splice the backing string and re-shift every later offset,
  and some mutations are simply disallowed.

**Special vs. non-special schemes.** The standard hardcodes a set of "special"
schemes with distinct parsing rules (default ports, mandatory authority, `\`
treated as `/`). Non-special schemes (`mailto:`, `data:`, custom) parse
differently and are frequently **cannot-be-a-base** URLs — they have no
manipulable path. `path_segments_mut()` and `join()` behave differently or
return `Err(())` on those; this is the most common source of confusion.

**Sub-crates carry the weight.** `percent-encoding` implements the per-component
encode sets (a query byte is escaped differently than a path byte).
`form_urlencoded` handles `application/x-www-form-urlencoded`. `idna` implements
UTS-46 / Punycode host processing and, since the 1.0 line, delegates its Unicode
data to a pluggable backend via `idna_adapter` — ICU4X by default, with
compile-time-lighter alternatives selectable through Cargo features.[^2]

## Production Notes

- **Parsing normalizes; do not expect round-trips.** `Url::parse(s).as_str() == s`
  is often false. Storing user-supplied URLs verbatim requires keeping the
  original string separately from the parsed `Url`.
- **`cannot-be-a-base` footgun.** For `data:`, `mailto:`, `javascript:` and
  similar, `path_segments()` returns `None` and `path_segments_mut()`/some
  `join()` calls return `Err`. Guard with `url.cannot_be_a_base()` before
  manipulating the path.
- **Scheme setter restrictions.** `set_scheme` refuses transitions between
  special and non-special schemes (e.g. `https` → `mailto`) and returns `Err(())`
  rather than silently corrupting the URL. Same for `set_host` on URLs that
  cannot have a host.
- **IDNA cost is transitive.** Even ASCII-only workloads pay the `idna`/ICU4X
  dependency in build time, MSRV, and binary size. Embedded / size-sensitive
  builds should read the `idna_adapter` README and opt into a smaller Unicode
  backend, or reconsider whether they need full IDNA at all.
- **Not RFC 3986.** Systems that must interoperate byte-for-byte with an RFC 3986
  peer (some signing schemes, canonical-request builders for cloud APIs) can get
  subtly different results from a WHATWG parser. Validate against the peer's
  exact rules rather than assuming equivalence.
- **`serde` support is opt-in** behind the `serde` feature; it serializes the URL
  as its string form.
- Pure parsing library: no network, no DNS, no TLS. `Url` guarantees syntactic
  validity, never reachability.

## When to Use / When Not

**Use when:**
- You are building or consuming HTTP(S) URLs and want the same normalization a
  browser applies (comparison, deduplication, fetching).
- You need correct IDNA / Punycode host handling for internationalized domains.
- You want relative-reference resolution (`join`) that matches the web platform.
- You are already transitively depending on it (most HTTP stacks) — reuse it.

**Avoid when:**
- You need strict RFC 3986 / 3987 semantics or byte-exact round-tripping.
- You are size- or MSRV-constrained and cannot absorb the ICU4X-backed `idna`
  dependency (embedded, tiny WASM).
- You only need to percent-encode/decode — depend on `percent-encoding` directly.

## Alternatives

- hyperium/http — its `Uri` type is RFC 3986-flavored, lighter, and native to the
  `hyper`/`tower` stack. Use when you live in that ecosystem and want no IDNA.
- yescallop/fluent-uri — RFC 3986/3987 URI/IRI parser, `no_std`, minimal deps.
  Use when you need standards-strict, byte-preserving parsing without ICU4X.
- servo/rust-url (`percent-encoding` sub-crate) — use directly when you only need
  encode/decode, not full URL parsing.
- The `iri-string` crate — use when you specifically need RFC 3987 IRI
  conformance and allocation control rather than WHATWG behavior.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2013-12 | Started inside the Servo project.[^1] |
| url 1.0 | 2016 | First 1.x line; earlier WHATWG-tracking API. |
| url 2.0 | 2019 | Major API cleanup; dropped deprecated 1.x surface.[^4] |
| idna 1.0 | 2024 | Pluggable Unicode backend via `idna_adapter`, ICU4X default.[^2] |
| url 2.5.x | 2024–2026 | Current line; ICU4X-backed IDNA, active maintenance. |

## References

[^1]: WHATWG URL Standard — the living spec `url` implements. https://url.spec.whatwg.org/
[^2]: `idna_adapter` crate docs — selecting a Unicode back end and its size/MSRV tradeoffs. https://docs.rs/crate/idna_adapter/latest
[^3]: `url` crate API documentation (`Url` type and component accessors). https://docs.rs/url/
[^4]: UPGRADING.md — migration notes between major versions. https://github.com/servo/rust-url/blob/main/UPGRADING.md

## Tags

rust, url-parser, whatwg-url, idna, percent-encoding, web, parsing, networking, servo, crates-io
