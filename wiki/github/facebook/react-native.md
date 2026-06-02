# facebook/react-native

The React-based framework for building native iOS and Android apps from a JavaScript/TypeScript codebase — the cross-platform mobile path most JS-first teams take.

## What it is

Originally open-sourced by Meta in 2015, React Native compiles a React component tree into native UI views (UIView on iOS, View on Android) rather than HTML elements. JavaScript drives the app's logic via a JS engine; the renderer translates virtual-DOM-style updates into native widget calls. The modern era is centered on the New Architecture (TurboModules + Fabric renderer) which removes the legacy bridge, plus the framework-driven flow where the default is to build via Expo rather than vanilla React Native.

## Key features

- Native UI rendering — components map to UIKit / Android Views, not WebViews.
- Hot reload + Fast Refresh in development.
- New Architecture (TurboModules + Fabric + JSI) — synchronous JS-to-native calls without the legacy async bridge.
- React Native for Web shares component code across mobile and web.
- Massive third-party ecosystem (`react-native-` packages on npm).
- Expo as the supported meta-framework that handles native builds, OTA updates, and library management.
- MIT-licensed.

## Tech stack

- C++ primary (Hermes JS engine, JSI, native renderer glue).
- JavaScript / TypeScript at the application layer.
- Per-platform native shells (Java/Kotlin on Android, Objective-C/Swift on iOS).
- Hermes (Meta's optimizing JS engine for mobile) is the default JS runtime as of recent releases.

## When to reach for it

- Your team is JS/TS-first and you want to ship mobile without staffing native iOS + Android engineers.
- You want true native UI fidelity (not a WebView wrap) with cross-platform code sharing.
- You're already on React for web and want maximum skill transfer to mobile.

## When *not* to reach for it

- You need bleeding-edge native API access on day one — wrapping new native APIs takes time.
- You want maximum runtime performance for graphics-heavy apps (games, intensive animations) — go native or use Flutter.
- You're allergic to Meta-stewardship — though the Expo-driven path mitigates this in practice.

## Maturity signal

125k stars, 25k forks, MIT, last push the day this page was generated. 11-year-old project with Meta + Microsoft (RN for Windows/macOS) + community contributors. The New Architecture migration is the project's biggest engineering effort in years; backwards-compat keeps the rollout gradual. Open-issues count of 1,283 tracks the breadth of platform-specific quirks; per-platform fixes are the dominant maintenance burden.

## Alternatives

- `flutter/flutter` — use when you want pixel-identical UI across platforms via custom rendering rather than native widgets.
- KMP (Kotlin Multiplatform) — use when you want shared business-logic with platform-native UI.
- SwiftUI + Jetpack Compose separately — use when maximum platform-native fidelity matters.
- Capacitor / Ionic — use when you want WebView-based cross-platform with web-skill reuse.

## Notes

The Expo recommendation matters in practice — most new React Native projects today start with Expo, not bare React Native. The New Architecture is gradually replacing the legacy bridge but the migration is multi-year. License (MIT) makes the codebase safe for commercial use; the binary licensing of bundled native libraries (Hermes, etc.) is also permissive.

## Tags

react-native, javascript, typescript, react, mobile-development, cross-platform, ios, android, framework, c-plus-plus, hermes
