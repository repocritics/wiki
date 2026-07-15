# ohler55/oj

> A C-extension JSON parser and Ruby object marshaller — historically the fastest JSON gem for MRI, built around swappable compatibility modes.

[GitHub repo](https://github.com/ohler55/oj) ·
[Official website](http://www.ohler.com/oj) ·
[License: MIT](https://github.com/ohler55/oj/blob/develop/LICENSE)

## Overview

Oj ("Optimized JSON") is a Ruby gem by Peter Ohler, first published in 2012[^1]. It is implemented as a native C extension and was for most of the 2010s the fastest way to parse and generate JSON on MRI Ruby, comfortably ahead of the standard library `json` gem and of `yajl-ruby`. It ships two distinct jobs in one library: a plain JSON parser/serializer, and a Ruby *object marshaller* that can round-trip arbitrary Ruby objects to and from a JSON-shaped encoding.

The defining design choice is **modes**. Rather than one JSON dialect, Oj exposes a `:mode` option (`:strict`, `:null`, `:compat`/`:json`, `:rails`, `:object`, `:custom`, `:wab`) that changes what the same `Oj.dump`/`Oj.load` calls do — from strict RFC-8259 JSON, to bug-for-bug imitation of the stdlib `json` gem, to imitation of Rails/ActiveSupport encoding, to Oj's own object-serialization format[^2]. This flexibility is the library's main value and its main hazard: behavior depends heavily on which mode is active, and the wrong default silently changes output.

The central tension in 2026 is that Oj's original selling point — raw speed — has eroded. The stdlib `json` gem was substantially rewritten and is now a fast C (and Java) extension in its own right[^3], so the margin that once justified adding a dependency is much smaller for plain parsing. Oj remains relevant mainly for its object/custom marshalling, its `Oj::Parser` and `Oj::Doc` fast paths, and codebases that already depend on it.

## Getting Started

```
gem install oj
```

```ruby
require 'oj'

h = { 'one' => 1, 'array' => [true, false] }

json = Oj.dump(h, mode: :strict)   # => '{"one":1,"array":[true,false]}'
back = Oj.load(json, mode: :strict) # => {"one"=>1, "array"=>[true, false]}

# Newer, faster, per-instance parser (no global option leakage)
p = Oj::Parser.new(:usual)
p.symbol_keys = true
p.parse(json)                       # => {:one=>1, :array=>[true, false]}
```

Because Oj is a C extension it compiles on install and requires a build toolchain (or a precompiled platform gem). It targets CRuby/MRI; it is not usable on JRuby, and TruffleRuby support is best-effort.

## Architecture / How It Works

Oj is C against the MRI object API. Parsing does not go through a Ruby tokenizer; it walks the byte buffer directly and constructs Ruby objects (or invokes callbacks) as it goes. Several parse strategies coexist:

- **`Oj.load` / `Oj.dump`** — the classic entry points. Behavior is governed by the global default options merged with per-call options and the `:mode`. Convenient, but the reliance on process-global defaults is the historical footgun.
- **`Oj::Parser`** — introduced in the 3.13 line to isolate options per parser instance and offer faster, allocation-lean parsing. It comes with delegates: `:usual` (build Ruby objects), `:saj` (SAX-style event callbacks), and `:validate` (parse without building)[^4].
- **`Oj::Doc`** — opens a JSON document as an immutable tree and lets you fetch values by path (e.g. `doc.fetch('/array/1')`) without materializing the whole structure as Ruby objects. This is the design described in Ohler's "Need for Speed" write-up and is the fastest option for pulling a few fields out of large documents[^5].

The **modes** are the conceptual core:

- `:strict` — only JSON-native types; anything else raises.
- `:null` — like strict, but non-serializable values dump as `null`.
- `:compat` / `:json` — mimic the stdlib `json` gem, including its quirks.
- `:rails` — mimic ActiveSupport's JSON encoding; paired with `Oj.optimize_rails`.
- `:object` — Oj's internal marshalling format. Uses sentinel keys (`^o`, `^c`, `^i`, etc.) to encode class names, object references, and instance variables so Ruby objects round-trip.
- `:custom` — everything configurable; the most predictable mode for bespoke needs.
- `:wab` — a minimal, WAB-oriented type set.

`Oj.mimic_JSON` goes further: it replaces the stdlib `JSON` module so that `require 'json'` throughout an app routes to Oj. This is a global monkeypatch and interacts unpredictably with gems that depend on exact stdlib `json` behavior.

## Production Notes

- **`:object` mode on untrusted input is a remote-code-execution risk.** Because it instantiates arbitrary classes named in the payload, loading attacker-controlled JSON in `:object` (or permissive `:custom`) mode is a deserialization vulnerability in the same family as `Marshal.load` and `YAML.load`. Never use object mode for external data; keep external parsing in `:strict`/`:compat`. Oj documents this in its Security notes[^6].
- **Global default options bleed across the process.** `Oj.default_options = {...}` changes behavior for every caller, including gems you don't control. This has caused hard-to-trace bugs where one library's default flips another's output. `Oj::Parser` instances exist specifically to avoid this — prefer them in new code.
- **`mimic_JSON` / `optimize_rails` are load-order sensitive.** Enabling them changes serialization globally; subtle differences from stdlib `json` (key ordering, escaping of `/` and HTML-unsafe characters, `to_json` argument handling) surface in tests and API contracts. Verify golden output after enabling.
- **The speed advantage is smaller than folklore suggests.** Modern stdlib `json` closed much of the gap[^3]; benchmark on your own payloads before adding Oj purely for parse speed. Oj still tends to win on generation and on the `Oj::Doc`/`Oj::Parser` fast paths.
- **Native gem, native problems.** Compilation failures on new Ruby versions, musl/Alpine builds, and cross-compiled platform gems are the usual operational friction. Pin versions and rebuild native extensions on Ruby upgrades.
- **Float and bignum precision** are configurable (`:bigdecimal_load`, `float_precision`); defaults trade exactness for speed, which matters for financial data.

## When to Use / When Not

**Use when:**
- You need Ruby object marshalling to a JSON-shaped format (`:object`/`:custom`) — Oj's most differentiated feature.
- You extract a few fields from large JSON documents and can use `Oj::Doc`.
- You want the `:saj` streaming/event API for very large inputs without building full object trees.
- A Rails app needs a drop-in encoder via `Oj.optimize_rails` and you accept the global monkeypatch.

**Avoid when:**
- You only parse trusted JSON into hashes and are on a recent Ruby — stdlib `json` is fast enough and dependency-free.
- You run on JRuby or need guaranteed portability off CRuby.
- You cannot tolerate a C-extension build step in your deploy pipeline.
- You'd be tempted to use `:object` mode on data crossing a trust boundary.

## Alternatives

- flori/json — the Ruby stdlib `json` gem; C/Java backed and much faster than it once was. Use instead when you only need plain JSON and want zero extra dependencies.
- brianmario/yajl-ruby — bindings to the yajl streaming C parser. Use instead when incremental/streaming parse of a socket or huge file is the primary need.
- msgpack/msgpack-ruby — MessagePack, a binary serialization format. Use instead when you control both ends and want smaller, faster payloads than any JSON.
- intridea/multi_json — a thin adapter that abstracts over Oj/json/yajl. Use instead when a library must not hard-code a specific JSON backend.
- ohler55/ox — same author, the equivalent fast parser/marshaller for XML. Use instead when the wire format is XML, not JSON.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012 | Initial release; C-extension JSON parser and object marshaller[^1]. |
| 2.x | 2013–2015 | Mode maturation, Rails/`json` mimic paths, `Oj::Doc`. |
| 3.0 | 2017 | Mode system reworked; per-mode option isolation clarified[^2]. |
| 3.13 | 2021 | `Oj::Parser` added — instance-scoped options, faster parse, `:usual`/`:saj`/`:validate` delegates[^4]. |
| 3.x | through 2026 | Ongoing maintenance; repo active as of 2026 with clang-format tooling and CI on the `develop` branch. |

## References

[^1]: Oj on RubyGems — release history and gem metadata. https://rubygems.org/gems/oj
[^2]: Oj documentation, "Modes" — strict, null, compat, rails, object, custom, wab. https://github.com/ohler55/oj/blob/develop/pages/Modes.md
[^3]: `json` gem repository and changelog (performance rewrite). https://github.com/ruby/json
[^4]: Oj documentation, "Advanced" — `Oj::Parser` and delegates. https://github.com/ohler55/oj/blob/develop/pages/Advanced.md
[^5]: Peter Ohler, "Need for Speed" — `Oj::Doc` design. http://www.ohler.com/dev/need_for_speed/need_for_speed.html
[^6]: Oj documentation, "Security" — object-mode deserialization considerations. https://github.com/ohler55/oj/blob/develop/pages/Security.md

## Tags

ruby, json, json-parser, serialization, marshalling, c-extension, rubygem, rails, performance, deserialization-security
