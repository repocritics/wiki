# Genymobile/scrcpy

Display and control your Android device from a desktop — the canonical tool for mirroring an Android screen to a host computer, with mouse/keyboard input passthrough.

## What it is

A C-language tool that screencasts an Android device (over USB or TCP/IP) to a desktop window, with bidirectional input: typing on the host keyboard sends to the device, mouse clicks become taps. Uses Android's built-in screen-recording API plus ADB — no root or app installation needed on the phone. The "scrcpy" pronunciation is "screen copy". Cross-platform host: Linux, macOS, Windows.

## Key features

- USB or Wi-Fi connection to the Android device via ADB.
- Real-time mirroring at the host's display refresh rate.
- Bidirectional input — type on host, sends keystrokes; click on window, sends taps.
- Audio forwarding (Android 11+).
- Screen recording — saves a file directly without screen-capturing the host window.
- Multi-device support — connect and mirror multiple Androids in separate windows.
- No app install on the device — uses ADB + built-in Android screen-recording APIs.
- Apache 2.0 licensed.

## Tech stack

- C primary (core mirroring tool).
- FFmpeg for video encoding/decoding.
- SDL2 for the display window and input capture.
- Android's `MediaCodec` + `screenrecord` for device-side capture.

## When to reach for it

- You're an Android developer demoing your app on a desktop screen during reviews.
- You're an Android user who wants to type with a real keyboard while using the phone.
- You're recording bug reproductions and want a tool that captures the device screen cleanly.
- You're managing many devices and want a CLI-friendly mirror.

## When *not* to reach for it

- You're on iOS — scrcpy is Android-only. QuickTime Player on macOS or a paid tool for iOS instead.
- You want a polished GUI — scrcpy is CLI-first with optional GUI wrappers.
- You need remote-cloud-device farms — `scrcpy` is local-USB or local-network only.

## Maturity signal

143k stars, 13k forks, Apache 2.0, last push 2026-05-29 — actively maintained by Genymobile (the company behind the Android emulator with the same name). The 2,821 open-issues count tracks per-device quirks and Android-version compatibility. License + corporate stewardship + the project's age (8 years) make this the most stable Android-screen-mirroring choice.

## Alternatives

- Vysor — commercial alternative with a polished GUI; freemium model.
- AirDroid — use when you want a paid, hosted, browser-based Android-control surface.
- Android Studio's "Running Devices" panel — built-in screen mirroring inside the IDE.
- QuickTime / Reflector — for iOS, not Android.

## Notes

scrcpy's "no app install on the device" property is its most-cited advantage — it works with any Android device that has USB debugging enabled, without polluting the user's phone with a companion app. Apache 2.0 + Genymobile stewardship + the maintainer's responsiveness make this the right default for anyone who needs reliable Android mirroring across multiple devices.

## Tags

android, c, command-line-interface, mirroring, screen-recording, developer-tools, apache-license, cross-platform
