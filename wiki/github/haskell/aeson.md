# haskell/aeson

> The de facto JSON library for Haskell — typeclass-driven encoding/decoding built around a single `Value` type.

[GitHub repo](https://github.com/haskell/aeson) ·
[Hackage](https://hackage.haskell.org/package/aeson) ·
[License: BSD-3-Clause](https://github.com/haskell/aeson/blob/master/LICENSE)

## Overview

aeson is the standard JSON serialization library for Haskell. It was written
by Bryan O'Sullivan in 2011[^1] and has been the ecosystem default for over a
decade — most Haskell libraries that touch JSON (web frameworks, config
loaders, API clients) depend on aeson's `ToJSON` / `FromJSON` classes rather
than parsing JSON themselves. The name is a Greek-myth pun: Aeson was the
father of Jason.

The design centers on one intermediate type, `Value`, and two typeclasses.
`ToJSON a` turns an `a` into JSON; `FromJSON a` parses JSON into an `a`. You
almost never write these by hand for record types — GHC generics
(`deriveGeneric` + default methods) or Template Haskell (`deriveJSON`) generate
them. The defining tension is between that convenience and control: derived
instances encode Haskell field names and constructor tags in ways you often
need to override via `Options` (field/tag modifiers, `omitNothingFields`,
`sumEncoding`), and the derived code is the main source of both compile-time
cost and surprising wire formats.

aeson's second defining trait is performance-consciousness that occasionally
leaks into the API. It maintains a separate `Encoding` type so serialization
can go straight to a `ByteString` builder without materializing a `Value`, and
it has repeatedly reworked its parser and object representation for speed and
DoS resistance. Those reworks are the source of its most painful breaking
changes (see History).

## Getting Started

aeson is on Hackage; add it to your `.cabal` `build-depends` or use `cabal`:

```bash
cabal install --lib aeson
# or add `aeson` to build-depends in your .cabal / package.yaml
```

Deriving instances generically:

```haskell
{-# LANGUAGE DeriveGeneric #-}
import Data.Aeson
import GHC.Generics
import qualified Data.ByteString.Lazy as BL

data User = User { userId :: Int, name :: String }
  deriving (Show, Generic)

instance ToJSON User
instance FromJSON User

main :: IO ()
main = do
  BL.putStr (encode (User 1 "Tom"))          -- {"userId":1,"name":"Tom"}
  print (decode "{\"userId\":2,\"name\":\"Brad\"}" :: Maybe User)
```

Hand-written instances use the parser combinators when you need control:

```haskell
instance FromJSON User where
  parseJSON = withObject "User" $ \o ->
    User <$> o .: "userId"
         <*> o .:? "name" .!= "anonymous"   -- optional with default
```

## Architecture / How It Works

**The `Value` type** is a sum: `Object`, `Array`, `String`, `Number`, `Bool`,
`Null`. Three representation choices matter operationally:

- `Number` is a `Scientific` (arbitrary-precision, `coefficient * 10^exp`).
  This avoids silent `Double` precision loss but means a literal like
  `1e1000000000` describes a huge number cheaply — a historical DoS vector the
  library has hardened against (2.3.0 rejects exponents below -1024)[^2].
- `String` and object keys are `Text`.
- `Object` is a `KeyMap` keyed by `Key`. Before 2.0 this was a plain
  `HashMap Text Value`; the 2.0 rewrite introduced dedicated `Data.Aeson.Key`
  and `Data.Aeson.KeyMap` modules[^3].

**Two classes, two directions.** `ToJSON` has both `toJSON :: a -> Value` and
`toEncoding :: a -> Encoding`. The `Encoding` type is a bytestring builder that
skips the intermediate `Value`; implementing `toEncoding` is the standard way
to make serialization faster. `FromJSON`'s `parseJSON :: Value -> Parser a`
runs in the `Parser` monad, which accumulates JSONPath context so errors can
say *where* they failed.

**Decoding** was historically an attoparsec parser (`Data.Aeson.Parser`). aeson
2.2 replaced the default path with a new tokenizer-based decoder
(`Data.Aeson.Decoding`) that is faster and streams tokens rather than building
an attoparsec parse tree[^4]. `decodeStrictText` (2.2) decodes directly from
`Text`, avoiding a UTF-8 round-trip through `ByteString`.

**Deriving** happens two ways — GHC generics (no extra dependency, slower to
compile) and Template Haskell (`deriveJSON` et al., TH stage restrictions).
Both consume the same `Options` record, so wire format is configured
identically regardless of mechanism.

## Production Notes

**The 1.x → 2.0 migration is the big one.** Objects changed from
`HashMap Text Value` to `KeyMap Key`. Any code that called `Data.HashMap`
functions on an aeson `Object`, or that used `Text` keys directly, breaks.
The fix is to switch to `Data.Aeson.KeyMap` operations and `Data.Aeson.Key`
(`fromText` / `toText`). Libraries supporting both eras typically use CPP or
the `aeson-2` boundary in their `.cabal` version bounds.

**Hash-flooding DoS.** Because objects were hash-based, untrusted input with
many colliding keys could degrade to quadratic behavior. aeson 2.0 added an
`-ordered-keymap` build flag that backs `KeyMap` with an ordered structure to
bound worst-case cost — relevant if you decode attacker-controlled JSON.

**Number blowups.** A decoded `Value` can hold a `Scientific` whose
materialization to `Integer`/`Double` is huge. Validate exponent/precision
bounds before forcing untrusted numbers.

**Lazy vs strict decode.** `decode`/`eitherDecode` take a lazy `ByteString`;
`decodeStrict`/`eitherDecodeStrict` a strict one; `decode'` forces to WHNF.
Mismatching these against your input source causes surprise space leaks. Prefer
`eitherDecode*` — the `Maybe` variants discard the parse error.

**Wire-format drift on refactors.** Derived instances mirror Haskell field and
constructor names, so renaming a field silently changes the JSON you emit. Pin
the format with `fieldLabelModifier`/`constructorTagModifier` and an explicit
`sumEncoding`, and test the serialized bytes, not just round-trips. Note that
generic-derived instances for large types are also a known drag on GHC build
times; TH deriving compiles faster but adds stage-restriction friction.

## When to Use / When Not

**Use when:**
- You're doing anything with JSON in Haskell and want ecosystem
  interoperability — most libraries already speak `ToJSON`/`FromJSON`.
- You have record/ADT types and want derived instances with tunable output.
- You need fast whole-document encode/decode with a mature, battle-tested API.

**Avoid / supplement when:**
- Documents don't fit in memory or you need incremental/streaming parsing —
  aeson builds a full `Value` (reach for a streaming library).
- You want a single source of truth for both codec *and* JSON Schema — a
  bidirectional-codec library on top of aeson fits better than two hand-kept
  instances.
- Decode throughput is your bottleneck at scale — SIMD-backed bindings can beat
  aeson's pure-Haskell decoder.

## Alternatives

- velveteer/hermes — simdjson (SIMD) bindings; use when raw decode throughput
  on large documents dominates and you can accept a C++ dependency.
- ondrap/json-stream — incremental/streaming parser; use when documents are too
  large to hold as a `Value` in memory.
- NorfairKing/autodocodec — one bidirectional codec that yields encoder,
  decoder, and JSON Schema; use when you want the schema and instances to never
  drift apart. Builds on aeson.
- fumieval/deriving-aeson — `DerivingVia`-based option configuration; use when
  you want aeson's `Options` expressed as types instead of TH/generics config.
- qfpl/waargonaut — lens/decoder-combinator approach with precise control; use
  when you want explicit decoders over typeclass-driven ones.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0.0 | 2011 | Initial release by Bryan O'Sullivan[^1]. |
| 0.10.0.0 | 2015 | `Encoding` type / `toEncoding` — direct-to-bytestring serialization. |
| 1.0.0.0 | 2016 | Major API cleanup; generic + TH deriving consolidated. |
| 2.0.0.0 | 2021 | `Key`/`KeyMap` replace `HashMap Text`; `-ordered-keymap` flag[^3]. |
| 2.1.0.0 | 2022 | Incremental fixes and instance additions. |
| 2.2.0.0 | 2023 | New tokenizer decoder (`Data.Aeson.Decoding`); `omitNothingFields` rework[^4]. |
| 2.3.0.0 | 2026-05-21 | Reject fractional exponents below -1024; stricter number parsing[^2]. |
| 2.3.1.0 | 2026-07-05 | `Data.Complex`/`Data.Fixed` instances; doc fixes[^2]. |

## References

[^1]: aeson README — "originally written by Bryan O'Sullivan." https://github.com/haskell/aeson/blob/master/README.markdown
[^2]: aeson changelog (2.3.0.0 / 2.3.1.0 entries). https://github.com/haskell/aeson/blob/master/changelog.md
[^3]: `Data.Aeson.KeyMap` / `Data.Aeson.Key` — introduced in aeson 2.0. https://hackage.haskell.org/package/aeson/docs/Data-Aeson-KeyMap.html
[^4]: `Data.Aeson.Decoding` — Hackage module reference. https://hackage.haskell.org/package/aeson/docs/Data-Aeson-Decoding.html

## Tags

haskell, json, serialization, parsing, encoding, typeclass, library, template-haskell, ghc-generics, hackage
