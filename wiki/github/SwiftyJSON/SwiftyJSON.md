# SwiftyJSON/SwiftyJSON

> Dynamic, crash-safe traversal of untyped JSON in Swift — the pre-Codable workhorse that still fits schemaless data.

[GitHub repo](https://github.com/SwiftyJSON/SwiftyJSON) ·
[License: MIT](https://github.com/SwiftyJSON/SwiftyJSON/blob/master/LICENSE)

## Overview

SwiftyJSON is a single-type Swift library that wraps arbitrary parsed JSON (the `Any` you get back from `JSONSerialization`) in a `JSON` value you can subscript freely without optional-chaining pyramids or `as?` casts. It was created in June 2014, days after Swift itself was announced at WWDC[^1], and for the three years before Swift shipped `Codable` it was the de facto answer to "how do I read JSON in Swift" — it still carries ~23k stars and ~3.4k forks as a result.

The defining tension is that SwiftyJSON solves a problem the language later absorbed. `Codable` (Swift 4, 2017)[^2] gives you compile-time-checked, strongly-typed decoding for JSON with a known shape, which is what most apps actually have. SwiftyJSON does the opposite: it keeps everything dynamic and untyped, trading type safety for the ability to reach into arbitrary, deeply-nested, or schema-unstable JSON without declaring a model first. Reaching a missing key never crashes and never throws at the call site — it returns a null `JSON` whose typed accessors yield `nil` (or a zero value) plus an `.error`.

That makes the library a poor fit for the common case (stable API contracts → use `Codable`) and a good fit for the residual cases `Codable` handles awkwardly: exploratory parsing, heterogeneous arrays, config blobs, and passing loosely-structured JSON through a system without ever modeling it. As of 2026 the repo is still committed to (Swift 6 / Xcode 16 support lives on `master`) but has not cut a tagged release since 5.0.2 in March 2024[^3].

## Getting Started

Swift Package Manager (add to `Package.swift`):

```swift
dependencies: [
    .package(url: "https://github.com/SwiftyJSON/SwiftyJSON.git", from: "5.0.0"),
]
```

Also distributed via CocoaPods (`pod 'SwiftyJSON', '~> 5.0'`) and Carthage.

```swift
import SwiftyJSON

let json = try? JSON(data: dataFromNetwork)

// Optional getter: nil if the path is missing or the wrong type.
if let name = json?[0]["user"]["name"].string {
    print(name)
}

// Non-optional getter: "" / 0 / [] / [:] fallback instead of nil.
let followers = json?["user"]["followers_count"].intValue ?? 0

// Deep access never traps; the error is inspectable, not fatal.
let miss = json?[999]["wrong_key"]
print(miss?.error as Any)   // e.g. "Array[999] is out of bounds"
```

## Architecture / How It Works

The whole library is essentially one type. `JSON` is a value type that boxes the underlying `Any` from `JSONSerialization`, tagged by an internal `Type` (number, string, bool, array, dictionary, null, unknown). Every subscript returns another `JSON`, so `json["a"][0]["b"]` is a chain of value copies, each carrying either a real sub-node or a null node with an attached error — this is what makes traversal non-crashing.

Access comes in two flavors that are easy to confuse:

- **Optional getters** (`.string`, `.int`, `.bool`, `.array`, `.dictionary`) return `nil` when the node is absent or the wrong type.
- **Non-optional getters** (`.stringValue`, `.intValue`, `.arrayValue`, `.dictionaryValue`) coerce to a default (`""`, `0`, `[]`, `[:]`). These never signal failure, so a typo silently becomes an empty string.

Subscripting accepts `Int`, `String`, or a `[JSONSubscriptType]` path array, so `json[1, "list", 2, "name"]` and `json[path]` are equivalent to the chained form. The type conforms to the literal-convertible protocols (`ExpressibleByStringLiteral`, etc.), so `let j: JSON = ["a": 1]` works, and to `Sequence` for `for (key, subJSON) in json` iteration (keys are stringified indices for arrays).

Everything is eager and reference-nothing: `JSONSerialization` parses the entire payload into a Foundation object graph up front, and SwiftyJSON boxes it. There is no lazy parsing, no streaming, and no direct `Codable` bridge — you convert at the boundary (`JSON(data:)` in, `.rawData()` / `.rawString()` out). Error reporting moved to a `SwiftyJSONError` enum in 4.x (`indexOutOfBounds`, `wrongType`, `notExist`, `invalidJSON`, …)[^4].

## Production Notes

- **`Codable` is usually the right default now.** If your JSON has a schema, `JSONDecoder` gives you type checking, `Sendable` models, and no third-party dependency. Teams frequently keep SwiftyJSON around only for a handful of legacy or genuinely-dynamic call sites; new code paths rarely need it.
- **Non-optional getters hide bugs.** `.stringValue` on a mistyped key returns `""` with no crash and no thrown error, so schema drift and typos surface as empty data far downstream. Prefer the optional getters (`.string`) and check `.error` / `.exists()` when correctness matters.
- **Performance is the runtime-boxing tax.** Every access does a dynamic type check against `Any`, and the whole tree is materialized eagerly via `JSONSerialization`. For large payloads or hot loops this is measurably slower than a `Codable` struct decode; it is fine for typical response sizes, not for high-throughput parsing.
- **Not thread-safe for mutation.** The `JSON` value is mutable (setters, `merge(with:)`), and concurrent mutation of a shared instance is unsafe. Under Swift 6 strict concurrency, passing `JSON` across isolation domains is friction because it is not `Sendable`; treat instances as single-owner.
- **Release cadence has slowed.** 5.0.0 shipped in 2019 and 5.0.2 in 2024; `master` tracks newer toolchains (Swift 6 / Xcode 16) but you may need to pin to the branch rather than a tag to get the latest platform fixes[^3]. The API itself is stable and effectively feature-complete.
- **`Int` width and number precision** follow `NSNumber` semantics inherited from `JSONSerialization`; very large integers and exact-decimal money values carry the usual floating-point caveats — do not use `.doubleValue` for currency.

## When to Use / When Not

**Use when:**
- The JSON is genuinely schemaless, deeply nested, or heterogeneous and modeling it as structs is disproportionate.
- You're doing exploratory or throwaway parsing and want to reach a value in one line.
- You maintain an existing SwiftyJSON codebase and a full `Codable` migration isn't justified.

**Avoid when:**
- The response has a stable contract you can model — reach for `Codable` / `JSONDecoder` instead.
- You need Swift 6 `Sendable` guarantees or heavy concurrency around parsed data.
- Parsing is performance-critical or payloads are large.
- You want to minimize dependencies in a new project.

## Alternatives

- apple/swift-foundation — the built-in `Codable` / `JSONDecoder`; use it whenever the JSON maps to a fixed, known schema (the modern default).
- Flight-School/AnyCodable — decode heterogeneous or dynamic JSON while staying inside the `Codable` pipeline, when you want dynamism without a separate library.
- thoughtbot/Argo — functional-style typed decoding from the same era; now inactive, listed for legacy codebases only.
- bignerdranch/Freddy — typed JSON parsing contemporary with SwiftyJSON; also no longer actively developed.
- utahiosmac/Marshal — dictionary-to-model marshaling with typed extraction, an option if you want light typing without full `Codable` boilerplate.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2014-09-26 | First tagged release, months after Swift's debut[^1]. |
| 2.0.0 | 2014-10-08 | Early API stabilization for Swift 1.x. |
| 3.0.0 | 2016-09-22 | Swift 3 source overhaul. |
| 4.0.0 | 2017-11-08 | Swift 4; `SwiftyJSONError` enum replaces string error domains[^4]. |
| 5.0.0 | 2019-04-02 | Swift 5 support[^3]. |
| 5.0.2 | 2024-03-27 | Latest tagged release; maintenance fixes[^3]. |

## References

[^1]: Repository created 2014-06-18, immediately following Swift's announcement at Apple WWDC 2014. https://github.com/SwiftyJSON/SwiftyJSON
[^2]: Swift Evolution SE-0166/SE-0167, "Codable" — introduced in Swift 4 (2017). https://developer.apple.com/documentation/swift/codable
[^3]: SwiftyJSON releases — 5.0.0 (2019-04-02) through 5.0.2 (2024-03-27). https://github.com/SwiftyJSON/SwiftyJSON/releases
[^4]: SwiftyJSON README, "Error" — `SwiftyJSONError` cases (`indexOutOfBounds`, `wrongType`, `notExist`, `invalidJSON`). https://github.com/SwiftyJSON/SwiftyJSON#error

## Tags

swift, ios, json, json-parsing, cocoapods, carthage, swift-package-manager, dynamic-typing, codable-alternative, apple-platforms
