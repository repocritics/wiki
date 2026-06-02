# vercel/hyper

Hyper — an Electron-based terminal built on HTML/CSS/JS. Designed for extensibility via npm plugins.

## What it is

A TypeScript / Electron terminal emulator authored at Vercel (then Zeit). Built around the premise "what if terminal customization was as easy as npm install". Each user's config file is JS; plugins install via `hyper i plugin-name`. Trades raw performance for extensibility — Hyper is heavier than native terminals but easier to theme and modify.

## Key features

- Plugin system via npm.
- Theme via `hyper.js` config.
- Cross-platform (Windows, macOS, Linux).
- xterm.js as the terminal engine.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Electron + xterm.js.

## When to reach for it

- You want terminal customization via npm + JS config.
- You're already in the Vercel / Next ecosystem and want matching tooling.

## When *not* to reach for it

- You want maximum performance — Alacritty, WezTerm, Ghostty are faster.
- You want a lightweight terminal — Hyper is Electron-based, ships ~150MB.

## Maturity signal

Maintenance pace has slowed since the early peak; competing terminals (Warp, Ghostty, Alacritty) have taken focus.

## Alternatives

- Alacritty / Kitty / WezTerm / Ghostty for performance.
- iTerm2 (macOS) / Windows Terminal — platform-native.

## Tags

typescript, electron, terminal, cross-platform, mit-license, vercel, developer-tools
