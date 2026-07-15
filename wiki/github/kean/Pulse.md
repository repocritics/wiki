# kean/Pulse

> On-device network and log inspector for Apple platforms — a SwiftUI console you embed in your own app rather than a system proxy.

[GitHub repo](https://github.com/kean/Pulse) ·
[Official website](https://pulselogger.com) ·
[License: MIT](https://github.com/kean/Pulse/blob/main/LICENSE)

## Overview

Pulse is a logging and `URLSession` inspection framework for Apple platforms,
written by Alexander Grebenyuk (kean), the author of Nuke and Get. It records
network requests and log messages into a local store and renders them with
`PulseUI`, a set of SwiftUI views you integrate directly into your app. The
premise is that the debugger should live inside every test build: QA and
teammates inspect traffic on the device itself and attach exported logs to bug
reports, instead of needing a Mac and a proxy set up.[^1]

The defining constraint — and the source of most confusion — is that Pulse is
**not** a network proxy. It does not sit between the device and the network the
way Charles or Proxyman do; it observes `URLSession` from inside your process,
so it only sees traffic that flows through a session it has been wired into.[^2]
Requests made by `Network.framework` sockets, `WKWebView`, or third-party SDKs
with their own transport are invisible unless they too use `URLSession`. The
upside is that there is no TLS interception, no certificate to install, and logs
never leave the device.

Pulse ships as three products in one repository: the `Pulse` core framework
(store + capture), `PulseUI` (the SwiftUI console), and `PulseLogHandler` (a
`swift-log` backend). A separate macOS app, Pulse Pro, opens exported `.pulse`
documents and offers real-time remote viewing.[^3]

## Getting Started

Add via Swift Package Manager:

```swift
// Package.swift
.package(url: "https://github.com/kean/Pulse", from: "5.0.0")
```

Route a session through Pulse's proxy delegate so its requests are recorded:

```swift
import Pulse

let session = URLSession(
    configuration: .default,
    delegate: URLSessionProxyDelegate(),   // forwards to your own delegate too
    delegateQueue: nil
)
```

Then embed the console anywhere in your UI — typically behind a debug menu or a
shake gesture, gated to non-release builds:

```swift
import PulseUI
import SwiftUI

struct DebugMenu: View {
    var body: some View {
        ConsoleView()   // full-text search, filters, request/response inspector
    }
}
```

To capture logs as a `swift-log` backend instead, register `PersistentLogHandler`
from `PulseLogHandler` in your `LoggingSystem.bootstrap`.

## Architecture / How It Works

Everything funnels into a `LoggerStore` — by default `LoggerStore.shared`,
backed by Core Data over a SQLite file. Network events and log messages are
persisted as entities; the store enforces a size limit and sweeps old records so
it does not grow without bound. Exporting a store produces a `.pulse` file, which
is the interchange format Pulse Pro and the share sheet consume.[^4]

Network capture has two paths. The explicit path is `URLSessionProxyDelegate`,
which you install as your session's delegate; it records metrics and
request/response bodies while still forwarding callbacks to any delegate you
supply. The automatic path, `Experimental.URLSessionProxy`, registers a custom
`URLProtocol` so requests are captured without touching your session setup — but
it is namespaced `Experimental` for a reason: `URLProtocol` interception does not
compose cleanly with all configurations (background sessions, streaming uploads,
some third-party stacks) and can alter timing.[^2] Framework integrations such as
Alamofire and Get work because they run on `URLSession` underneath.

`PulseUI` is SwiftUI-only. `ConsoleView` and the network inspector are views you
compose; there is no UIKit-native console, so UIKit apps present them through
`UIHostingController`. The console reads directly from the `LoggerStore`, so
what you see is the same persisted data you can export.

## Production Notes

- **Bodies contain secrets.** The inspector shows full request/response bodies
  and headers, including `Authorization` tokens and cookies. Do not ship the
  console in App Store release builds without redaction; the common pattern is to
  compile `PulseUI` and the capture wiring only under `#if DEBUG`, or gate access
  behind an internal flag. Pulse can exclude or filter sensitive fields, but that
  is opt-in, not default.
- **It only sees `URLSession`.** Traffic through `WKWebView`, raw
  `Network.framework`, gRPC libraries, or SDKs bundling their own networking is
  not recorded. Teams expecting a full packet view are repeatedly surprised; for
  that you need an actual proxy.
- **Store growth and trimming.** The default store trims by size, so very old or
  very large payloads are evicted — Pulse is a rolling buffer, not an audit log.
  If you need durable capture, raise the limit deliberately and watch disk use.
- **SwiftUI floor.** Pulse 5 requires iOS 15 / macOS 12 / visionOS 1 and Swift
  5.10 (Xcode 15.4); Pulse 4 goes down to iOS 14. Projects on older SwiftUI or
  UIKit-heavy codebases pay an integration and binary-size cost for a debug tool.
- **The automatic proxy is experimental.** Prefer the explicit
  `URLSessionProxyDelegate` in production test builds; reserve
  `Experimental.URLSessionProxy` for quick spikes where you accept the caveats.
- **Pulse Pro is a separate app with its own history.** The macOS viewer has
  changed availability and pricing over the project's life; the stable, free
  interchange path is the `.pulse` export opened wherever it is supported.[^3]

## When to Use / When Not

**Use when:**
- You want testers to inspect and share network logs on-device without a Mac,
  proxy, or CA certificate.
- Your networking goes through `URLSession` (directly or via Alamofire/Get).
- You want a SwiftUI-native debug console and a `swift-log` sink in one library.

**Avoid when:**
- You need to see all device traffic, including WebViews and non-`URLSession`
  stacks — use a real proxy.
- You need remote, aggregated observability across a fleet — use a crash/APM SDK.
- You cannot afford to embed debug UI or accept the SwiftUI version floor.

## Alternatives

- proxyman/Proxyman — system-level HTTP(S) proxy; use when you need to intercept
  and rewrite all traffic (including WebViews) from outside the app.
- pmusolino/Wormholy — zero-integration in-app network debugger triggered by a
  shake; use when you want capture without wiring a framework or store.
- getsentry/sentry-cocoa — remote crash, performance, and network breadcrumbs;
  use when you want aggregated production telemetry rather than on-device viewing.
- CocoaLumberjack/CocoaLumberjack — general-purpose file/console logging; use when
  you need durable log output, not a network inspector.
- apple/swift-log — the logging API Pulse plugs into as a backend; use directly
  when you only need a logging facade and route it elsewhere.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2021-08 | Public launch: `URLSession` logging + SwiftUI console.[^1] |
| 2.0 | 2022 | Major rewrite; expanded store and network inspector. |
| 3.0 | 2023 | Console and store refinements, broader platform support. |
| 4.0 | — | Swift 5.7 / Xcode 14.1 baseline; iOS 14, tvOS/watchOS/macOS.[^5] |
| 5.0 | — | Swift 5.10 / Xcode 15.4; iOS 15 floor, adds visionOS 1.[^5] |

## References

[^1]: Alexander Grebenyuk, "Pulse" — kean.blog. https://kean.blog/pulse/home
[^2]: Pulse README, "Pulse is **not** a network proxy" and integration notes. https://github.com/kean/Pulse
[^3]: Pulse Pro / Pulse logger site. https://pulselogger.com
[^4]: Pulse documentation (LoggerStore, network logging). https://kean-docs.github.io/pulse/documentation/pulse/
[^5]: Pulse README, "Minimum Requirements" table. https://github.com/kean/Pulse

## Tags

swift, ios, macos, visionos, networking, logging, urlsession, swiftui, debugging, developer-tools, apple
