# tldraw/tldraw

tldraw — an infinite canvas whiteboard SDK. Embed Figma-style infinite canvas + shape editing into any web app. Direct competitor to Excalidraw with a stronger SDK story.

## What it is

A TypeScript React library that ships a polished infinite-canvas whiteboard component, plus an SDK for embedding + extending it. Used as both a destination (tldraw.com) and as embeddable infrastructure (e.g. for Vercel's AI-driven whiteboards). Distinguishes itself from Excalidraw via the SDK-first framing — tldraw is built to be embedded rather than only used standalone.

## Key features

- Infinite canvas with shapes, arrows, freehand drawing, sticky notes, text, images.
- Multiplayer / real-time collaboration via separate `@tldraw/sync` package.
- Embed as a React component into any app.
- Custom shape extension API.
- Asset import / paste support.
- License is TLDRAW SDK License (non-OSI commercial license with free tier).

## Tech stack

- TypeScript primary.
- React on the consumer side.

## When to reach for it

- You're embedding a polished whiteboard into your own product.
- You want a more SDK-flavored alternative to Excalidraw.

## When *not* to reach for it

- You want an Excalidraw-style hand-drawn aesthetic — Excalidraw fits better.
- You need a strict-OSI license — tldraw's license has commercial restrictions.

## Maturity signal

Actively maintained under tldraw Inc.

## Alternatives

- `excalidraw/excalidraw` — hand-drawn-style competitor.
- Miro, FigJam — commercial whiteboards.

## Tags

typescript, react, whiteboard, canvas, sdk, library, drawing, collaboration
