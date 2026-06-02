# flutter/flutter

Google's UI toolkit for building natively compiled cross-platform apps from a single Dart codebase — mobile (iOS / Android), desktop (Windows / macOS / Linux), and web.

## What it is

A cross-platform UI framework that compiles Dart code into native binaries for iOS, Android, macOS, Windows, Linux, and JavaScript-targeted web builds. Built on Skia (now Impeller) for direct GPU rendering, which gives Flutter its trademark consistency: the same widget tree looks pixel-identical across platforms because Flutter draws its own UI rather than wrapping native widgets. Material Design 3 and Cupertino design systems ship in-box.

## Key features

- Single Dart codebase compiles natively for six target platforms.
- Custom rendering engine (Impeller, replacing Skia) — Flutter draws every pixel rather than wrapping platform widgets.
- Hot reload during development — UI updates land in the running app in sub-second cycles.
- Material Design 3 + Cupertino design systems built-in with theming hooks.
- Strong package ecosystem on pub.dev — community + Google packages for state management, networking, storage, native interop.
- Native FFI for calling C/C++ libraries directly from Dart.
- Fuchsia support (Google's experimental OS) as a first-class target.

## Tech stack

- Dart at the application layer.
- C++ engine for rendering (Impeller / Skia) and platform glue.
- Per-platform shells (Java/Kotlin on Android, Swift/Objective-C on iOS, etc.) for the embedding layer.
- BSD-3-Clause licensed — the cleanest license among major cross-platform UI frameworks.

## When to reach for it

- You're building a mobile app that needs identical visual design across iOS and Android.
- You're a small team that can't maintain separate iOS + Android codebases.
- You need desktop + mobile from one codebase and want a single hire profile (Dart-only) to staff it.
- You're after pixel-perfect custom UI that doesn't need to match the host platform's native look.

## When *not* to reach for it

- You want first-class platform-native look — Flutter draws its own UI; Cupertino mimicry is close but not authentic.
- You're integrating heavily with platform-native UI (e.g. complex iOS share-sheet integrations, Android Auto). The platform-channel bridge adds friction.
- You're allergic to non-OSI alternative ecosystems — Dart is open-source but smaller than JS/Python/Swift.

## Maturity signal

176k stars, 30k forks, BSD-3-Clause, last push the morning this page was generated. 11-year-old project under Google stewardship with first-party support for six target platforms. The 12k open-issues count is high in absolute terms but proportional to the breadth: six platforms × ~10 years of accumulated edge cases. The project's release cadence remains tight and Material 3 support is current.

## Alternatives

- `facebook/react-native` — use when your team already lives in JS/TS and React, or you need closer-to-native iOS/Android components.
- KMP (Kotlin Multiplatform) — use when you want a shared business-logic layer with platform-native UI.
- `expo/expo` (React Native shell) — use when you want managed React Native with first-party tooling.
- SwiftUI + Jetpack Compose separately — use when you want maximum platform-native fidelity.

## Notes

The Impeller rendering migration (from Skia) was the project's biggest engine rewrite and is the reason for some of the recent open-issues count. Flutter's "draw every pixel" approach is the architectural bet: it's a feature for cross-platform consistency and a tax when integrating with native UI conventions. License (BSD-3-Clause) is the most permissive among major cross-platform UI toolkits.

## Tags

flutter, dart, cross-platform, mobile-development, ios, android, desktop, framework, user-interface, material-design
