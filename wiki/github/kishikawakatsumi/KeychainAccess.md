# kishikawakatsumi/KeychainAccess

> A Swift wrapper that makes Apple's Keychain Services usable without writing raw `SecItem` C calls.

[GitHub repo](https://github.com/kishikawakatsumi/KeychainAccess) ·
[License: MIT](https://github.com/kishikawakatsumi/KeychainAccess/blob/master/LICENSE)

## Overview

KeychainAccess is a thin Swift facade over Apple's Security framework (`SecItemAdd`,
`SecItemUpdate`, `SecItemCopyMatching`, `SecItemDelete`). The underlying API is a
CoreFoundation C interface driven by untyped `CFDictionary` attribute bags and
`OSStatus` integer error codes, widely considered one of the most awkward surfaces
in the Apple SDK. This library hides that: you get a `Keychain` object, subscript
and method access, a fluent options builder, and Swift `Error` values instead of
magic numbers. It targets iOS, macOS, watchOS, tvOS, and Mac Catalyst[^1].

First published in December 2014, it is one of the oldest and most-depended-on Swift
libraries still in wide use, and effectively the default Keychain wrapper in the
CocoaPods/SPM ecosystem. It is stable rather than active: the last commit to `master`
was mid-2024 and the API has changed little across the 4.x line[^2]. For a
security-primitive wrapper this quietness is a feature — the surface it wraps has
itself barely changed.

The defining tension is abstraction versus fidelity. Keychain has sharp, non-obvious
semantics — accessibility classes, iCloud sync namespaces, access-group entitlements,
biometric access control, items that survive app uninstall. KeychainAccess makes the
common path trivial but does not make those semantics go away, and its most convenient
affordance (subscripting) actively hides the errors that tell you when they bite.

## Getting Started

Swift Package Manager (`.package(url: "https://github.com/kishikawakatsumi/KeychainAccess.git", from: "4.2.2")`),
CocoaPods (`pod 'KeychainAccess'`), or Carthage.

```swift
import KeychainAccess

let keychain = Keychain(service: "com.example.github-token")

// Convenient but error-swallowing:
keychain["access-token"] = "01234567-89ab-cdef"
let token = keychain["access-token"]        // nil on miss OR on error

// Explicit and throwing — prefer this in production:
do {
    try keychain.set("01234567-89ab-cdef", key: "access-token")
    let value = try keychain.get("access-token")
    try keychain.remove("access-token")
} catch {
    print(error)   // KeychainAccess.Status wrapping the OSStatus
}
```

## Architecture / How It Works

A `Keychain` is a value type holding an options struct — service or server,
optional access group, accessibility class, synchronizable flag, authentication
policy, label/comment. Every operation converts that struct plus the key into a
`CFDictionary` query and calls the matching `SecItem*` function, translating the
returned `OSStatus` into a thrown `Status` error on failure.

Two item classes are exposed. `Keychain(service:)` maps to
`kSecClassGenericPassword` (application passwords); `Keychain(server:protocolType:)`
maps to `kSecClassInternetPassword` (URL-scoped credentials with protocol and
authentication-type attributes). These are distinct Keychain namespaces — an item
written under one class is invisible to a query under the other.

The fluent builder (`.accessibility(_:)`, `.synchronizable(_:)`, `.label(_:)`,
`.accessibility(_:authenticationPolicy:)`) returns a copy of the `Keychain` with the
option set, so configuration is immutable and chainable. `set` transparently does
add-or-update — it attempts `SecItemAdd` and on `errSecDuplicateItem` falls back to
`SecItemUpdate` — so callers never see the duplicate-item code that trips up
hand-rolled Keychain code.

Biometric protection is built on `SecAccessControl` and `LAContext`. Passing an
`authenticationPolicy` (e.g. `[.biometryAny]`) attaches an access-control object;
reading such an item triggers the Touch ID / Face ID prompt synchronously inside the
`SecItem` call, blocking the calling thread until the user responds — hence the
library's insistence that biometric operations run off the main thread.

Subscripting is sugar over the throwing methods with the error discarded: string and
data subscripts return `nil` both when an item is absent and when the call fails.
This is the single most important thing to understand about the API's shape.

## Production Notes

- **Subscript access hides failures.** `keychain["k"]` returns `nil` for a genuine
  miss and for `interactionNotAllowed` (screen locked, item is `whenUnlocked`),
  biometric cancellation, missing entitlements, and every other error. Anywhere the
  distinction between "no value" and "couldn't read" matters, use `try keychain.get`.
- **Keychain items survive app uninstall on iOS.** A fresh install can read
  credentials left by a previous one. For a clean first launch, clear the Keychain
  when a "has launched" flag is absent from `UserDefaults` (which *is* wiped on
  uninstall).
- **Default accessibility is `.afterFirstUnlock`.** The library defaults to
  `kSecAttrAccessibleAfterFirstUnlock`, not the bare `SecItem` default of
  `whenUnlocked` — usually right for background apps, but items stay readable while
  the device is locked (after first post-boot unlock) unless you pick a stricter class.
- **iCloud sync is a separate namespace.** `synchronizable(true)` sets
  `kSecAttrSynchronizable`; a query without that flag won't match a synchronized item
  and vice versa, so mixing synced and non-synced access to a key silently returns
  "not found."
- **Access groups require entitlements.** `OSStatus -34018` shows up when the Keychain
  Sharing capability is absent — common in app extensions, unit-test hosts, and some
  simulator configurations[^1].
- **Biometric reads block the thread.** The auth UI is presented from inside the
  synchronous `SecItem` call; run these off the main queue or the UI freezes while the
  prompt is up. There is no `async` accessor — the whole API is blocking.
- **macOS behaves differently.** The classic file-based login keychain and the
  data-protection keychain diverge on access-control and sharing semantics; iOS
  behavior is not guaranteed identical on macOS, especially for access groups.

## When to Use / When Not

**Use when:**
- You store small secrets (tokens, passwords, keys) on Apple platforms and don't want
  to touch `SecItem` directly.
- You want biometric-gated storage without hand-building `SecAccessControl`.
- You need access-group sharing or iCloud Keychain sync with a readable API.

**Avoid when:**
- You want zero dependencies and are comfortable with the Security framework — the
  wrapper is thin enough to inline the parts you use.
- You need to store large blobs — the Keychain is for small secrets, not general
  encrypted storage; use the file system with Data Protection instead.
- You need a maintained, actively evolving dependency — this one is effectively in
  maintenance mode, which is acceptable for a stable primitive but worth a deliberate
  decision.

## Alternatives

- evgenyneu/keychain-swift — smaller, simpler wrapper; use when you only need
  basic string get/set/delete and want minimal surface.
- square/Valet — opinionated safe-by-default Keychain library; use when you want
  accessibility and sharing decisions made for you.
- Apple Security framework (`SecItem*`) — no dependency, full control; use when you
  need exact control over queries or want no third-party code in a security path.
- jrendel/SwiftKeychainWrapper — older lightweight wrapper; legacy projects only.

## History

Dates below are approximate; the OS/Swift mapping is from the project's requirements
table[^1], exact tags on the releases page[^2].

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2014–2015 | Initial Swift 1.x wrapper; generic + internet password classes. |
| 2.0 | ~2015 | Swift 2; watchOS 2 support added. |
| 2.2 | ~2016 | tvOS 9 support. |
| 3.0 | ~2016 | Swift 3 migration. |
| 3.1–3.2 | ~2018 | Swift 4.x, then Swift 5.0 compatibility. |
| 4.0 | ~2019 | Swift 5.1. |
| 4.1 | ~2019 | Mac Catalyst 13 support[^1]. |
| 4.2.x | 2020–2024 | Maintenance line; current release, low churn[^2]. |

## References

[^1]: KeychainAccess README — features, requirements table (OS/Swift per version),
and the `-34018` entitlement note. https://github.com/kishikawakatsumi/KeychainAccess#readme
[^2]: KeychainAccess releases and commit history (last commit to `master` mid-2024).
https://github.com/kishikawakatsumi/KeychainAccess/releases

## Tags

swift, ios, macos, keychain, security, credentials, secitem, biometrics, touch-id, apple, cocoapods
