# react/create-react-app

Create React App (CRA) — the historical "one command sets up a React project" tool. Now deprecated; modern guidance points to Vite, Next.js, or Remix.

## What it is

A JavaScript CLI tool that scaffolded a new React project with webpack + Babel + a development server preconfigured. Was the canonical recommendation in the React docs from 2016 through 2022. Officially deprecated in 2023; the React team now directs new users to framework-based scaffolds (Next.js, Remix) or Vite-based starters.

## Key features

- Single-command scaffolding (`npx create-react-app my-app`).
- Zero-config webpack setup with hot reloading + production builds.
- Eject command for full customization (one-way, irreversible).

## Tech stack

- JavaScript primary.
- Webpack-based build (legacy).

## When to reach for it

- You're maintaining an existing CRA-based project.
- You're studying the historical React tooling landscape.

## When *not* to reach for it

- You're starting a new React project — use Vite + React, Next.js, or Remix instead.
- You need active maintenance — CRA is in long-tail mode.

## Maturity signal

103k stars, 27k forks, MIT — historically essential. Officially deprecated in 2023. Open issues count of 2,405 reflects the long tail of breakage as the underlying webpack/Babel stack drifts; no plans to fix.

## Alternatives

- Vite + React (`pnpm create vite`) — modern lightweight scaffold.
- Next.js (`create-next-app`) — for full-stack React.
- Remix — alternative full-stack React framework.

## Tags

react, javascript, build-tool, scaffolding, legacy, mit-license, deprecated
