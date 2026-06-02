# expo/expo

The meta-framework on top of React Native — universal apps for Android, iOS, and web from one codebase, with managed builds, OTA updates, and a curated library set.

## What it is

A framework + tooling layer on top of React Native that handles the "native builds, signing, asset management" pain that bare RN leaves to the developer. Expo Router brings Next.js-style file-based routing to RN; EAS Build + EAS Update handle CI builds and over-the-air updates. The "managed workflow" is the default; "bare workflow" still ejects to vanilla RN when needed. Modern React-Native development effectively starts with Expo, not RN itself.

## Key features

- `npx create-expo-app` scaffolds projects with sensible defaults.
- Expo Router — file-system-based routing across iOS, Android, web.
- EAS (Expo Application Services) for build, submit, update, ATS hosting.
- Expo SDK — curated set of native APIs (camera, location, sensors, notifications, etc.) typed and consistent.
- Universal apps — same components render on iOS, Android, and Web.
- MIT-licensed.

## Tech stack

- TypeScript primary across the SDK, CLI, and tooling.
- React Native underneath.
- Hermes JS engine on device.

## When to reach for it

- You're starting a new RN project — Expo is the modern default.
- You want to ship to iOS, Android, AND web from one codebase with shared components.
- You need OTA app updates without going through app-store releases.

## When *not* to reach for it

- You have unusual native-module needs that Expo's managed workflow doesn't cover.
- You're allergic to vendor dependencies — Expo + EAS adds vendor-lock-in beyond bare RN.

## Maturity signal

50k stars, 13k forks, MIT, actively maintained under Expo Inc. Open-issues count of 602.

## Alternatives

- Bare React Native — use when you need full control.
- Flutter — alternative cross-platform framework.
- T3-stack-flavored web-only — use when mobile isn't your target.

## Tags

react-native, typescript, mobile-development, ios, android, cross-platform, framework, mit-license, expo
