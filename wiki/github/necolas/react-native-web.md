# necolas/react-native-web

> A DOM implementation of React Native's component and API surface, so the same component code renders as native views on mobile and as HTML on the web.

[GitHub repo](https://github.com/necolas/react-native-web) ·
[Official website](https://necolas.github.io/react-native-web) ·
[License: MIT](https://github.com/necolas/react-native-web/blob/master/LICENSE)

## Overview

React Native for Web (RNW) is a reimplementation of React Native's primitives — `View`, `Text`, `Image`, `ScrollView`, `Pressable`, `StyleSheet`, `Animated`, etc. — that targets `react-dom` instead of native iOS/Android views. It was created by Nicolas Gallagher while at Twitter, where it powered the Twitter Lite mobile web app shipped in 2017[^1]. The premise is "write once against the React Native API, run on native and web," and its most consequential downstream role today is as the web renderer inside Expo[^2].

The defining tension is that RNW is not a port of React Native — it is a parallel implementation of RN's public API that maps onto the DOM. That means it tracks React Native's surface rather than being generated from it, so newer RN APIs can lag, and web-only concerns (semantic HTML, hover, focus rings, responsive CSS) are bolted onto an API that was designed for a mobile touch runtime. In practice you get genuine code reuse for layout and logic, but the web output is a tree of `<div>`s styled with generated CSS, not idiomatic HTML — a real cost for SEO-sensitive or accessibility-strict sites unless you work at it.

The project has never reached a 1.0 release; it has lived in the `0.x` range for its entire history and is effectively maintained by a single author at a deliberate, slow cadence[^3]. It remains widely deployed because Expo depends on it, not because of a large contributor base.

## Getting Started

```bash
npm install react-native-web react-dom
# with a bundler alias so `react-native` resolves to `react-native-web`
npm install --save-dev babel-plugin-react-native-web
```

```jsx
// App.js — identical source can run on native RN and on web
import { View, Text, StyleSheet, Pressable } from "react-native";

export default function App() {
  return (
    <View style={styles.box}>
      <Text style={styles.label}>Hello from both platforms</Text>
      <Pressable onPress={() => console.log("pressed")}>
        <Text>Tap me</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  box: { padding: 16, alignItems: "center" },
  label: { fontSize: 18, fontWeight: "600" },
});
```

On web you additionally mount with `AppRegistry.runApplication` (or Expo's entry) and configure your bundler to alias `react-native` → `react-native-web`. Expo and Next.js integrations wire this up for you.

## Architecture / How It Works

RNW is delivered as a monorepo whose primary published package is `react-native-web`[^4]. Consuming apps set a module alias (`react-native` → `react-native-web`) via Metro, webpack, Babel, or Vite so that imports written against `react-native` resolve to the web implementation at build time. This aliasing is the load-bearing mechanism — there is no runtime interop layer between native and web.

Core pieces:

- **Component primitives** — each RN component is reimplemented as a React component that renders DOM elements. `View` becomes a `<div>` with flexbox defaults matching Yoga's layout model (RN uses `flexDirection: column` by default, and RNW replicates that), `Text` becomes a semantic-ish span/div, `Image` wraps `<img>` with RN's `resizeMode` semantics.
- **StyleSheet → atomic CSS** — `StyleSheet.create` objects are compiled to atomic CSS classes and injected into the document. Since the `0.11` rewrite the style system deduplicates declarations into single-property utility classes and applies them by class name, which keeps stylesheet size bounded as the app grows[^3]. This is closer to a CSS-in-JS engine than to inline styles.
- **Accessibility mapping** — RN accessibility props (`accessibilityRole`, `accessibilityLabel`, etc.) are translated to ARIA attributes and, where possible, appropriate DOM roles. Gallagher has treated a11y as a first-class concern, but it is still a mapping, not native HTML semantics.
- **Event/interaction layer** — RN's responder system (`Pressable`, touch events) is emulated on top of DOM pointer/mouse/keyboard events, adding web-only affordances like hover and focus that don't exist in the native API.

The coupling that matters most is with **React Native's own API evolution** and with **Expo**. RNW must chase RN's public surface; when RN ships a new prop or component, RNW implements it separately, so parity is best-effort. Expo pins specific RNW versions per SDK, which in practice makes Expo's release train the real compatibility authority for most users.

## Production Notes

**It is a `0.x` project that thousands of apps depend on via Expo.** The version number understates its maturity but accurately reflects its willingness to make breaking changes between minor releases. Pin the version and upgrade deliberately; do not float `^0.19`.

**Output is `<div>` soup, not semantic HTML.** SEO, first-meaningful-paint, and screen-reader quality all require explicit effort. If you need clean semantic markup or server-rendered content-first pages, RNW fights you; teams with those needs often keep a separate web codebase rather than share components.

**SSR is possible but not free.** RNW supports server rendering (used by frameworks like Expo Router / Next.js integrations), but style extraction on the server, hydration mismatches, and the cost of shipping the RN API shim to the browser are real. Bundle size is meaningfully larger than a hand-written react-dom app for equivalent UI.

**Third-party RN libraries are the biggest footgun.** RNW only implements React Native core. Any native module (camera, maps, gesture-handler, reanimated) either needs a web-compatible build or must be conditionally excluded. `react-native-gesture-handler` and `react-native-reanimated` ship web builds; many community libraries do not, and you discover this at bundle time. `.web.js` / `.native.js` platform-extension files are the standard escape hatch.

**Styling parity gaps.** Not every web CSS capability is expressible through RN's StyleSheet subset, and not every RN style prop maps cleanly to CSS. Media queries, `:hover`, and complex selectors live outside the RN model and require RNW-specific escape hatches or a companion CSS layer.

**Maintenance cadence is slow and centralized.** Issue and PR throughput reflects a single primary maintainer. Do not assume a fast turnaround on bug fixes; budget for carrying local patches on large deployments.

## When to Use / When Not

**Use when:**
- You are building with Expo and want your mobile app to also run on the web with shared component code.
- Your product is an app-like SPA (dashboards, tools, logged-in experiences) rather than a content/SEO site.
- You want to maximize code reuse across iOS, Android, and web and accept web output that is app-shaped.

**Avoid when:**
- The web target is a content-first, SEO-critical, or accessibility-strict site — semantic HTML frameworks serve you better.
- You are web-only and have no native app; use `react-dom` (or a React framework) directly rather than emulating the RN API.
- You depend heavily on native modules without web builds, or you need cutting-edge RN APIs the moment they land.

## Alternatives

- expo/expo — if you want the managed universal-app path; Expo bundles and version-pins RNW for you, so most people should adopt RNW through Expo rather than wiring it by hand.
- facebook/react-native — use directly when you only ship native and don't need a web target at all.
- tamagui/tamagui — use when you want universal (native + web) UI with a compiler that optimizes styles and produces leaner web output than RNW's runtime.
- nandorojo/solito — use when you want to share screens between a Next.js web app and a React Native app via a navigation bridge rather than aliasing everything to RNW.
- facebook/react (react-dom) — use when the web is your only platform; skip the RN abstraction entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial release | 2015 | Repo created; early DOM implementation of RN primitives[^4]. |
| Twitter Lite | 2017 | RNW ships in production powering Twitter's mobile web PWA[^1]. |
| 0.11 | 2019 | Major rewrite: atomic-CSS style system, performance and API overhaul[^3]. |
| 0.18 | 2022 | React 18 compatibility line. |
| 0.19 | 2022–2023 | Current major line; the version range Expo SDKs pin against. |

Version dates beyond the `0.11` rewrite are approximate; consult the GitHub releases for exact tags.

## References

[^1]: Nicolas Gallagher, "Twitter Lite and High Performance React Progressive Web Apps at Scale" — 2017. https://medium.com/@nicolasgallagher/making-twitter-lite-and-high-performance-react-progressive-web-apps-at-scale-d28a00e780a3
[^2]: Expo documentation, "Develop for web." https://docs.expo.dev/workflow/web/
[^3]: React Native for Web documentation and release notes. https://necolas.github.io/react-native-web/docs/
[^4]: necolas/react-native-web repository and `packages/react-native-web`. https://github.com/necolas/react-native-web

## Tags

javascript, react, react-native, react-dom, cross-platform, css-in-js, ui-framework, web, expo, accessibility, monorepo
