# marmelroy/PhoneNumberKit

> A Swift framework for parsing, formatting, and validating international phone numbers — a native port of Google's libphonenumber metadata and rules.

[GitHub repo](https://github.com/marmelroy/PhoneNumberKit) ·
[License: MIT](https://github.com/marmelroy/PhoneNumberKit/blob/master/LICENSE)

## Overview

PhoneNumberKit is the de-facto phone-number library for Swift on Apple platforms. It parses arbitrary phone-number strings into a validated `PhoneNumber` value, formats numbers to the E.164, international, and national styles, and ships an as-you-type formatter plus a UIKit text field and country-code picker. Its correctness comes from bundling the metadata of Google's libphonenumber project[^1] — the same country rules, number-type patterns, and example numbers — rather than reimplementing validation heuristics from scratch. It was created and maintained for years by Roy Marmelstein.

The defining tension of this repository as of 2026 is that **it is frozen**. The README states development has moved to a new GitHub organization: the maintained core lives at `PhoneNumberKit/PhoneNumberKit` (5.0.0+) and the UI pieces at `PhoneNumberKit/PhoneNumberKitUI`, while `marmelroy/PhoneNumberKit` is pinned at `4.3.0` and no longer receives metadata refreshes[^2]. Because validation accuracy depends entirely on how fresh the bundled libphonenumber metadata is, a frozen copy silently rots: new number ranges and country changes are not reflected, so a valid number can be rejected or a retired range accepted. The repo is not marked archived on GitHub and still shows recent pushes, but the README's "no longer maintained" notice is the operative signal.

## Getting Started

Swift Package Manager is the preferred distribution channel; CocoaPods and Carthage are also supported.

```swift
// Package.swift — point new projects at the maintained org fork
dependencies: [
    .package(url: "https://github.com/PhoneNumberKit/PhoneNumberKit", from: "5.0.0")
]
```

```swift
import PhoneNumberKit

// PhoneNumberUtility parses the bundled metadata on init and holds it for
// the object's lifetime — allocate ONCE and reuse, it is expensive.
let util = PhoneNumberUtility()

do {
    let number = try util.parse("+44 20 7031 3000")   // region auto-detected
    number.countryCode        // 44
    number.nationalNumber     // 2070313000
    number.type               // .fixedLine

    util.format(number, toType: .e164)          // +442070313000
    util.format(number, toType: .international)  // +44 20 7031 3000
} catch {
    print("invalid phone number")
}
```

## Architecture / How It Works

The library is built around a single stateful object, historically named `PhoneNumberKit` and renamed to `PhoneNumberUtility` in the 4.x line. On initialization it loads and decodes the bundled `PhoneNumberMetadata` JSON — the libphonenumber ruleset covering every country's national number patterns, mobile/fixed-line/toll-free distinctions, formatting templates, and example numbers. This decode is the reason allocation is costly and the README warns to hold one instance rather than allocating per-parse.

`parse(_:withRegion:ignoreType:)` normalizes the raw string, extracts the country calling code (or applies the supplied/derived default region), matches the national number against the metadata patterns, and by default runs a **hard type validation** that classifies the number (mobile, fixed line, etc.). That type check is the expensive part of parsing; `ignoreType: true` skips it when you only need a syntactically valid number. A batched `parse([String])` path exists for validating large arrays quickly and silently drops invalid entries.

`PhoneNumber` is an immutable struct (`countryCode`, `nationalNumber`, `numberExtension`, `type`, `numberString`). Formatting is a separate concern handled by `format(_:toType:)`. Interactive input is served by `PartialFormatter` (the as-you-type engine) and `PhoneNumberTextField`, a `UITextField` subclass that formats on the fly and optionally shows a flag, example-number placeholder, and country-code prefix. `CountryCodePickerViewController` provides the selectable country list, themeable via `CountryCodePickerOptions`.

The UIKit coupling is the structural line the new org draws: core parsing/formatting is platform-agnostic Swift, but the text field, picker, and their UIKit dependencies were split into a separate `PhoneNumberKitUI` package in the 5.x reorganization, so headless/server or SwiftUI-only consumers can take the core without dragging in UIKit.

## Production Notes

- **Freshness is the whole ballgame.** Validation is only as correct as the embedded metadata. On the frozen `marmelroy` 4.3.0 build, metadata no longer updates — migrate to `PhoneNumberKit/PhoneNumberKit` for any app where a wrongly-rejected number is a real user-facing bug (signup, SMS OTP, contact import).
- **Allocation cost.** `PhoneNumberUtility()` parses metadata on every init. Allocating per-call (e.g., inside a table-cell formatter or a tight validation loop) is a common and measurable performance mistake; inject a single shared instance instead.
- **Parse cost vs. type validation.** The README quotes ~0.4 s per 1000 parses; the hard type check dominates that. For bulk validation where you only need "is this shaped like a real number," pass `ignoreType: true` and/or use the array-parse API.
- **App binary size.** The bundled libphonenumber JSON adds non-trivial weight to the app; there is no build-time option to strip unused regions, so you ship the full global ruleset even for a single-country app.
- **UIKit-only UI.** `PhoneNumberTextField` and the picker are UIKit. SwiftUI users wrap them in `UIViewRepresentable`; there is no first-party SwiftUI field. This is unchanged by the org move (the UI just lives in a separate package).
- **Migration is low-friction.** The module name `PhoneNumberKit` is unchanged in the org fork, so core imports need no source changes; UI users add the `PhoneNumberKitUI` package and a second `import`[^2].

## When to Use / When Not

**Use when:**
- You need libphonenumber-grade parsing/validation/formatting natively in Swift on iOS/macOS without bridging to C++ or shelling out to a service.
- You want a ready-made as-you-type text field and country picker for a signup or contact form.
- You validate or normalize user-entered numbers to E.164 before storing or sending them.

**Avoid when:**
- You are starting fresh — depend on `PhoneNumberKit/PhoneNumberKit` (5.0.0+) instead of this frozen repo, so metadata stays current.
- You need cross-platform parity with a backend that validates the same numbers — align both sides on the same libphonenumber version; a stale Swift copy will disagree with a fresh server copy.
- Your app is extremely size-sensitive and needs single-region validation only — the full bundled metadata may be more than you want to ship.

## Alternatives

- google/libphonenumber — the upstream reference (Java/C++/JS/…); use when you need the authoritative implementation, a non-Apple platform, or server-side validation that must exactly match clients.
- PhoneNumberKit/PhoneNumberKit — the maintained continuation of this project (5.0.0+, UI split into PhoneNumberKitUI); use this for any new Swift project instead of the frozen `marmelroy` repo.
- iziz/libPhoneNumber-iOS — an Objective-C port of libphonenumber; use when you need Objective-C interop or a closer line-for-line port of the original.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2015 | First public release; Swift port of libphonenumber metadata and rules[^1]. |
| 4.x | 2024 | Primary type renamed `PhoneNumberKit` → `PhoneNumberUtility`; SPM the preferred distribution. |
| 4.3.0 | 2025 | Final release on `marmelroy/PhoneNumberKit`; repository frozen[^2]. |
| 5.0.0 | 2026 | Maintenance moves to the `PhoneNumberKit` org; core/UI split into two packages[^2]. |

## References

[^1]: Google libphonenumber — the metadata and validation rules PhoneNumberKit ports. https://github.com/google/libphonenumber
[^2]: PhoneNumberKit README migration notice — repository frozen at 4.3.0, development moved to the PhoneNumberKit organization. https://github.com/marmelroy/PhoneNumberKit

## Tags

swift, ios, phone-number, validation, formatting, parsing, libphonenumber, e164, uikit, cocoapods, swift-package-manager
