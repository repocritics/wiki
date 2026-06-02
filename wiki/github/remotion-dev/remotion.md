# remotion-dev/remotion

Make videos programmatically with React — compose video frames as React components, render to MP4.

## What it is

A TypeScript framework that lets developers define video content as React components: each frame is rendered by React, then ffmpeg encodes the sequence into MP4 / GIF / WebM. Used for procedurally-generated YouTube content, data-driven explainers, social-media video templates, and bulk video personalization. Commercial license required for some use cases (verify SPDX `NOASSERTION`).

## Key features

- React components → video frames.
- Composition system with timeline, audio, transitions.
- ffmpeg-based encoding.
- Browser preview + headless render via `remotion render`.
- Lambda render for cloud-scale parallel rendering (paid).
- License is `NOASSERTION` — see Remotion's specific commercial-license requirements.

## Tech stack

- TypeScript primary.
- React on the composition side.
- ffmpeg for encoding.

## When to reach for it

- You're generating videos programmatically (data viz, automated explainers, personalized video at scale).
- You want React skills to transfer to video production.

## When *not* to reach for it

- You want a video-editor UI — DaVinci Resolve, Premiere, Final Cut.
- You're allergic to commercial-licensing complexity — Remotion's license requires payment for some commercial uses.

## Maturity signal

49k stars, 3.4k forks, last push recent. Open issues = 127. Commercial license is the recurring caveat.

## Alternatives

- ffmpeg directly — for non-React programmatic rendering.
- Motion Canvas — alternative animation-via-code library.

## Tags

react, typescript, video, multimedia, framework, rendering
