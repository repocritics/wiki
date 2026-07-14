# swiftlang/swift

> The Swift compiler and standard library — a memory-safe systems language born at Apple, now governed as a language-agnostic open-source project.

[GitHub repo](https://github.com/swiftlang/swift) ·
[Official website](https://swift.org) ·
[License: Apache-2.0](https://github.com/swiftlang/swift/blob/main/LICENSE.txt)

## Overview

Swift is a statically typed, compiled language first announced by Apple at WWDC 2014 and open-sourced in December 2015[^1]. This repository is the compiler (`swiftc`), the standard library, and the core runtime — not the SDKs or IDE. The language was designed to replace Objective-C for Apple platform development while remaining a general-purpose systems language: value semantics by default, automatic reference counting instead of a tracing garbage collector, protocol-oriented generics, and memory safety enforced at compile time. The repo moved from the `apple` org to the vendor-neutral `swiftlang` org in 2024 as part of a broader governance shift toward the Swift open-source ecosystem[^2].

The defining tension in Swift is **Apple-platform gravity versus cross-platform ambition**. The language, compiler, and standard library are genuinely portable and officially support Linux and Windows; the toolchain builds and runs there. But the highest-value libraries developers actually reach for — SwiftUI, UIKit, Foundation's richer corners, Combine — historically shipped inside Apple's closed SDKs, and the tooling experience off Apple platforms remains rougher. The 2020s pushed hard against this: an open-source Foundation rewrite (swift-foundation), Swiftly toolchain management, and first-class Windows/Linux CI. The gap has narrowed but not closed.

Swift's other structural fact is that it is **large and slow to build**. The compiler is a multi-hundred-thousand-line C++ codebase built on LLVM, and its type checker performs bidirectional inference with operator overload resolution that can be superlinear in pathological expressions. This shapes everything from CI cost to the "expression too complex to type-check" errors that Swift developers periodically hit.

## Getting Started

On macOS, Swift ships inside Xcode. On Linux or Windows, install a toolchain via Swiftly[^3] or download from swift.org:

```bash
# Linux / macOS — Swiftly toolchain manager
curl -O https://download.swift.org/swiftly/linux/swiftly-$(uname -m).tar.gz
tar zxf swiftly-*.tar.gz && ./swiftly init
swiftly install latest
```

Create and run a package with the Swift Package Manager:

```bash
mkdir hello && cd hello
swift package init --type executable
swift run
```

A minimal program showing value types, optionals, and protocol-oriented generics:

```swift
struct User {
    let id: Int
    let name: String
}

protocol Greetable {
    var displayName: String { get }
}

extension User: Greetable {
    var displayName: String { name }
}

func greet<T: Greetable>(_ item: T) -> String {
    "Hello, \(item.displayName)"
}

let users = [User(id: 1, name: "Tom"), User(id: 2, name: "Brad")]
for user in users {
    print(greet(user))
}
```

## Architecture / How It Works

`swiftc` is a compilation pipeline layered on LLVM[^4]:

1. **Parse** — source → AST.
2. **Semantic analysis** — name binding and the constraint-based **type checker**, which solves a system of constraints to resolve overloads, generic parameters, and implicit conversions. This is the stage responsible for both Swift's terse inference and its worst compile-time blowups.
3. **SIL (Swift Intermediate Language)** — a Swift-specific IR unique to this compiler. SIL is where ARC optimization, generic specialization, devirtualization, and definite-initialization analysis happen. It is the single most important internal concept for understanding Swift performance.
4. **IRGen / LLVM** — SIL → LLVM IR → machine code via the LLVM backend.

**Memory management** is Automatic Reference Counting (ARC), not garbage collection. The compiler inserts retain/release calls at compile time; there is no runtime pause, but reference cycles leak unless broken with `weak`/`unowned`. This is a deliberate tradeoff: predictable, deterministic deallocation at the cost of programmer discipline around cycles.

**Value semantics.** `struct` and `enum` are value types with copy-on-write for the standard library's collections. This eliminates whole categories of shared-mutable-state bugs but means "everything is a class" mental models from Java/C# transfer poorly.

**Concurrency.** Swift's structured concurrency (`async`/`await`, `Task`, actors) shipped in Swift 5.5 (2021)[^5]. Actors provide data-race isolation enforced by the compiler. Swift 6 (2024) made this isolation checking strict by default under the Swift 6 language mode — the largest and most disruptive change in the language's history.

**Generics** are implemented via runtime witness tables (like Rust trait objects) rather than monomorphization by default, though the optimizer specializes hot paths. This keeps binary size down but can cost performance in un-specialized generic code.

The compiler, standard library, and SIL optimizer are tightly co-developed in this one repository; SwiftPM, Foundation, and testing frameworks live in sibling `swiftlang` repos.

## Production Notes

**Compile times are the dominant operational complaint.** The type checker's constraint solver can degrade badly on long expressions — chained ternaries, large literal collections, or heavy operator overloading — producing the notorious "the compiler is unable to type-check this expression in reasonable time" error. The fix is almost always to break the expression up and add explicit type annotations to help the solver.

**ABI stability is macOS/iOS-only.** Swift achieved ABI stability on Apple platforms with Swift 5.0 (2019), meaning the runtime ships in the OS and apps don't bundle the standard library[^6]. On Linux and Windows there is no stable ABI; binaries are tied to the toolchain version and the runtime is bundled. Cross-platform library authors cannot assume ABI stability.

**Swift 6 migration is a real project, not a version bump.** Enabling the Swift 6 language mode turns data-race warnings into errors. Codebases with global mutable state, non-`Sendable` types crossing concurrency boundaries, or completion-handler-based APIs require meaningful refactoring. Most teams adopt it incrementally by enabling upcoming-feature flags per module rather than flipping the whole codebase at once.

**Linux/Windows caveats.** The toolchain works, but expect fewer batteries: no SwiftUI/UIKit, a Foundation that historically lagged the Apple implementation (being addressed by swift-foundation), and a smaller pool of tested third-party packages. Server-side Swift (Vapor, Hummingbird) is viable but a niche compared to Go/Node/JVM.

**Long-term source compatibility is strong within a major version;** the compiler runs a source-compatibility suite of real open-source packages against every change. Cross-major migrations (Swift 3→4, and the 5→6 concurrency shift) are the exceptions that required code changes.

## When to Use / When Not

**Use when:**
- You are building for Apple platforms (iOS, macOS, watchOS, tvOS, visionOS) — it is the native, first-party language.
- You want memory safety and predictable (ARC, no-GC) performance in application code.
- You value compiler-enforced data-race safety for concurrent code (actors, `Sendable`).
- You want a modern type system (algebraic enums, protocols with associated types, generics) with strong tooling on macOS.

**Avoid when:**
- Your target is primarily Linux server infrastructure with an existing Go/Rust/JVM team — the ecosystem depth isn't there yet.
- You need fast iteration on a large codebase and cannot absorb Swift's compile times.
- You are writing bare-metal embedded firmware — Embedded Swift exists but is young relative to Rust's `no_std` ecosystem.
- You need a stable cross-platform ABI or guaranteed library parity outside Apple's platforms.

## Alternatives

- rust-lang/rust — memory safety without a garbage collector or ARC, stronger cross-platform and embedded story; choose it when you are not tied to Apple platforms and want compile-time ownership guarantees.
- JetBrains/kotlin — the closest analog for the opposite ecosystem (Android/JVM); choose it when your primary target is Android or the JVM rather than Apple platforms.
- google/go — simpler language, far faster builds, GC-based; choose it for networked backend services where developer velocity beats fine-grained performance control.
- apple/swift-corelibs-foundation / swiftlang/swift-foundation — the cross-platform Foundation implementation; relevant when evaluating Swift specifically for Linux/Windows.
- llvm/llvm-project — the backend Swift compiles through; relevant if you are working at the codegen or toolchain layer beneath Swift itself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-09 | First release, Xcode 6, Apple platforms only. |
| — | 2015-12-03 | Open-sourced under Apache-2.0; Linux port[^1]. |
| 3.0 | 2016-09 | Large source-breaking API redesign; SwiftPM introduced. |
| 4.0 | 2017-09 | Source compatibility modes, `Codable`. |
| 5.0 | 2019-03 | ABI stability on Apple platforms[^6]. |
| 5.5 | 2021-09 | Structured concurrency: `async`/`await`, actors[^5]. |
| 5.7 | 2022-09 | Regex literals, `if let` shorthand. |
| — | 2024 | Repo moved from `apple` to `swiftlang` org[^2]. |
| 6.0 | 2024-09 | Swift 6 language mode: strict data-race safety by default. |

## References

[^1]: Apple, "Swift is now open source" — 2015-12-03. https://www.swift.org/blog/swift-now-open-source/
[^2]: Swift.org, "The next chapter in Swift open source governance" — on the move to the swiftlang GitHub organization. https://www.swift.org/blog/next-steps-supporting-swift-community/
[^3]: Swiftly — the Swift toolchain installer and manager. https://www.swift.org/install/
[^4]: Swift compiler architecture documentation. https://github.com/swiftlang/swift/blob/main/docs/README.md
[^5]: Swift Evolution, "Concurrency" roadmap (SE-0296 async/await, SE-0306 actors) — Swift 5.5. https://www.swift.org/documentation/concurrency/
[^6]: Swift.org, "ABI Stability and More" — Swift 5.0, 2019. https://www.swift.org/blog/abi-stability-and-more/

## Tags

swift, apple, compiler, systems-programming, llvm, memory-safety, ios, macos, concurrency, arc, static-typing, cross-platform
