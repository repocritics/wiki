# greensock/GSAP

> Framework-agnostic JavaScript animation engine — a high-speed property tweener with a plugin ecosystem for scroll, SVG, and physics-grade motion.

[GitHub repo](https://github.com/greensock/GSAP) ·
[Official website](https://gsap.com) ·
[License: GreenSock Standard "No Charge" (source-available, not OSI)](https://gsap.com/standard-license)

## Overview

GSAP (GreenSock Animation Platform) is a JavaScript animation library that began life as ActionScript tweening for Adobe Flash (TweenLite/TweenMax, copyright dating to 2008) and was ported to JavaScript as Flash declined[^1]. It is not a rendering framework and not tied to the DOM: at its core it is a property manipulator that interpolates numeric values over time with sub-pixel precision, and it will animate anything JavaScript can reach — CSS, SVG attributes, canvas, WebGL uniforms, plain objects, colors, strings. The DOM/CSS handling is itself an optional plugin (CSSPlugin) that ships bundled with core.

The defining tension is **licensing versus capability**. For most of its history GSAP's most-wanted plugins (SplitText, MorphSVG, ScrollSmoother, DrawSVG, and others) were paywalled behind a "Club GreenSock" membership, while the core and a handful of free plugins used a custom "no charge" license. In October 2024, following Webflow's acquisition of GreenSock, the entire toolset — including every formerly members-only plugin — was made free for commercial use[^2]. This removed the single biggest historical objection to GSAP, but the license is still a bespoke GreenSock document, not an OSI-approved open-source license; the repository reports no SPDX license for that reason.

GSAP's constituency is interaction designers, agencies, and product teams building scroll-driven marketing sites, complex sequenced UI motion, and SVG/canvas effects where the declarative-per-component model of React animation libraries becomes awkward. GreenSock reports usage on over 12 million sites[^1]; whether or not that figure is auditable, GSAP is the incumbent for "serious" web animation work.

## Getting Started

```bash
npm install gsap
```

Or via CDN:

```html
<script src="https://cdn.jsdelivr.net/npm/gsap@3/dist/gsap.min.js"></script>
```

```javascript
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";

gsap.registerPlugin(ScrollTrigger); // plugins must be registered before use

// A timeline: sequenced, retimeable, reversible
const tl = gsap.timeline({ defaults: { duration: 0.6, ease: "power2.out" } });
tl.to(".box", { x: 200, rotation: 45 })
  .to(".box", { backgroundColor: "#e63946" }, "-=0.2") // overlap prev by 0.2s
  .from(".label", { autoAlpha: 0, y: 20 });

// Scroll-linked scrub animation
gsap.to(".panel", {
  xPercent: -100,
  ease: "none",
  scrollTrigger: { trigger: ".wrap", scrub: true, pin: true, end: "+=2000" },
});
```

## Architecture / How It Works

GSAP is organized as a small core plus registered plugins:

- **Core** exposes the tween factories (`gsap.to`, `gsap.from`, `gsap.fromTo`, `gsap.set`) and the `Timeline`. Every tween is a node on a global ticker driven by a single `requestAnimationFrame` loop; GSAP does not spin up a timer per animation. Values are read once, interpolated in JS, and written each frame — it does not delegate to native CSS transitions or the Web Animations API.
- **Timelines** are the differentiator versus most competitors: they are nestable, seekable, reversible playheads. A timeline is itself tweenable, so `tl.timeScale(2)` or `tl.reverse()` retimes an entire sequence, and relative position labels (`"-=0.2"`, `"<"`, `">"`) express sequencing without manual delay math.
- **Plugins** register themselves onto the core and hook the property-processing pipeline. CSSPlugin (bundled) parses transform strings, handles vendor prefixing, and manages the transform matrix so multiple tweens can target `x`/`rotation`/`scale` independently. ScrollTrigger, Draggable, Flip, MotionPathPlugin, MorphSVGPlugin, SplitText, and Observer are separate modules registered via `gsap.registerPlugin(...)`.
- **ScrollTrigger** is effectively its own subsystem: it batches scroll/resize reads to avoid layout thrashing, manages pinning by injecting spacer elements, and links a scroll position to a tween's progress (`scrub`). Most GSAP production complexity lives here rather than in core.

The npm package ships ES modules; a `/dist/` directory carries UMD builds for older toolchains. `gsap/all` re-exports the free plugins for convenience. `@gsap/react` adds a `useGSAP()` hook that wraps `useLayoutEffect` and auto-reverts animations on unmount, solving React's double-invoke and cleanup problems.

## Production Notes

- **The license is not open source.** GSAP is free (including commercially) but governed by GreenSock's custom "no charge" license, not MIT/Apache. Organizations with policies that require OSI-approved licenses, or that vendor dependencies through license scanners, will see GSAP flagged as "no license" / "unknown" and may need a manual exception. Do not assume MIT because the code is on GitHub.
- **GitHub issues are misleading as a health signal.** The repo shows only a handful of open issues because GreenSock routes support and bug reports to its own forums, not the issue tracker[^3]. Low issue count here does not mean low activity; conversely, do not expect fast GitHub-issue turnaround.
- **Bundle size and tree-shaking.** Core plus ScrollTrigger is non-trivial weight for a landing page. Import only the plugins you register; avoid `gsap/all` in production if you use a few plugins. Note the README's standing claim that most ad networks exclude GSAP from ad-size budgets — relevant only for banner-ad contexts.
- **React / SSR cleanup.** Without `useGSAP()` (or manual `gsap.context()` + `revert()`), animations leak across Strict Mode double-mounts and route changes, and DOM measurements taken before hydration are wrong. Always run GSAP in a layout effect, never in render.
- **`will-change` and pinning cost.** ScrollTrigger `pin: true` reflows the document by inserting spacers; heavy pinning plus many scrubbed timelines can jank on low-end mobile. Prefer transform-only animations (`x`, `y`, `scale`, `autoAlpha`) over layout properties (`top`, `width`) to stay on the compositor.
- **Accessibility.** GSAP does not respect `prefers-reduced-motion` automatically. Use `gsap.matchMedia()` to gate motion on that media query; this is opt-in, and omitting it is a common accessibility defect in GSAP-heavy sites.
- **Performance claims are vendor claims.** The "up to 20× faster than jQuery" line is GreenSock marketing; real-world frame budget depends far more on what properties you animate (compositor vs. layout) than on the library's inner loop.

## When to Use / When Not

**Use when:**
- You need sequenced, retimeable, reversible motion (timelines) rather than one-shot transitions.
- You're building scroll-driven storytelling / pinned sections — ScrollTrigger is the de facto standard.
- You need SVG morphing, motion paths, text splitting, or FLIP layout animations with one consistent API.
- You want the same animation engine across React, Vue, Svelte, vanilla, canvas, and WebGL.

**Avoid when:**
- Your org mandates OSI-approved licenses and can't grant an exception.
- You want declarative, component-local, physics-based animation inside React — a React-native library fits the mental model better.
- Your needs are simple hover/enter transitions that CSS transitions or the Web Animations API cover with zero dependencies.
- You're shipping an extreme-budget page where even the GSAP core is too much weight.

## Alternatives

- motiondivision/motion — declarative, React-first (Framer Motion) with a smaller vanilla core; use when your animations map cleanly to component state and you want an MIT license.
- juliangarnier/anime — lightweight MIT tweening library; use when you want timelines-lite without GSAP's size or license.
- pmndrs/react-spring — spring/physics-based React animation; use when interruptible, natural motion tied to state matters more than precise timelines.
- airbnb/lottie-web — renders After Effects animations exported as JSON; use when designers author motion in AE rather than in code.
- Native Web Animations API (no repo) — zero-dependency, browser-built-in; use when you only need keyframe/transition-grade motion and can drop scroll/SVG/morph features.

## History

| Version | Date | Notes |
|---------|------|-------|
| TweenLite/Max (AS) | 2008 | Origins as ActionScript tweening for Flash[^1]. |
| JS port | ~2011–2012 | GreenSock ported the engine to JavaScript as Flash waned. |
| 1.x / 2.x | 2014–2018 | TweenLite/TweenMax/TimelineLite era; separate classes. |
| 3.0 | 2019-11 | Unified `gsap` object, single `gsap.to()` API, ~50% smaller core, backward-compatible[^4]. |
| 3.x + ScrollTrigger | 2020-onward | ScrollTrigger plugin release; scroll animation becomes GSAP's flagship use case. |
| Free-for-all | 2024-10 | Webflow acquires GreenSock; entire toolset incl. all bonus plugins made free[^2]. |
| 3.15.x | 2025–2026 | Current v3 line; actively but not daily maintained (last push 2026-04). |

## References

[^1]: GSAP README and gsap.com — "high-speed property manipulator … over 12 million sites." https://github.com/greensock/GSAP
[^2]: Webflow, "GSAP is now free, including all of its bonus plugins" — 2024-10. https://webflow.com/blog/gsap-becomes-free
[^3]: GSAP support model — bug reports and help routed to the GreenSock forums rather than GitHub issues. https://gsap.com/community/
[^4]: GreenSock, "GSAP 3 Release Notes" — unified API and migration guide. https://gsap.com/resources/3-release-notes/

## Tags

javascript, animation, web-animation, scrolltrigger, svg, frontend, ui-motion, browser, framework-agnostic, source-available
