# dcloudio/uni-app

> Write once in Vue, ship to iOS, Android, HarmonyOS, web, and a dozen Chinese mini-program platforms.

[GitHub repo](https://github.com/dcloudio/uni-app) ·
[Official website](https://uniapp.dcloud.io) ·
[License: Apache-2.0](https://github.com/dcloudio/uni-app/blob/uni-app-x/LICENSE)

## Overview

uni-app is a cross-platform application framework from DCloud (a Beijing company), built on Vue.js syntax. One codebase compiles to native iOS/Android apps, HarmonyOS, responsive web, and — the reason it dominates in China — the many incompatible mini-program (小程序) runtimes: WeChat, Alipay, Baidu, ByteDance/Douyin, QQ, Kuaishou, Feishu, JD, and others[^1]. It also targets quick-apps (快应用) and HarmonyOS meta-services. Outside China the framework is little-known; inside China it is one of the default answers to "how do I ship the same product as a WeChat mini-program and a native app."

The project now ships in two lineages. **uni-app** (the original) is pure front-end tech: a JavaScript logic layer plus a web-view rendering layer, using the same runtime architecture as WeChat mini-programs. **uni-app x** (the `uni-app-x` branch, now the repo default) is a rewrite around UTS — a TypeScript-like language that compiles to Kotlin on Android, Swift on iOS, ArkTS on HarmonyOS Next, and JS on web/mini-programs — paired with a native `uvue` rendering engine[^2]. uni-app x trades the web-view compatibility layer for genuinely native rendering and closer-to-native performance.

The defining tension is ecosystem gravity. uni-app is fastest and best-documented inside DCloud's own world — the HBuilderX IDE, the uni_modules plugin market, uniCloud serverless backend, uni-id/uni-push. It is a Vue framework whose component model, styling unit (`rpx`), and page config (`pages.json`) are shaped by the mini-program target rather than by the browser or by Vue conventions, and whose primary documentation is Chinese. That is a good fit for teams targeting the Chinese super-app landscape, and an awkward one for teams that just want a Vue-based React Native alternative.

Note: GitHub reports the repo's dominant language as Objective-C — that reflects the checked-in iOS native runtime, not how apps are written. Application code is Vue + JS/TS (uni-app) or Vue + UTS (uni-app x).

## Getting Started

Two supported paths: the HBuilderX GUI (DCloud's IDE, where the framework is most integrated) or the CLI.

```bash
# Vue 3 + Vite CLI scaffold
npx degit dcloudio/uni-preset-vue#vite my-app
cd my-app
npm install
npm run dev:h5        # or dev:mp-weixin, dev:app, etc.
```

```vue
<!-- pages/index/index.vue -->
<template>
  <view class="content">
    <text>{{ title }}</text>
    <!-- #ifdef MP-WEIXIN -->
    <button @click="share">WeChat share</button>
    <!-- #endif -->
  </view>
</template>

<script setup>
import { ref } from 'vue'
const title = ref('Hello uni-app')
function share() { uni.showToast({ title: 'shared' }) }
</script>

<style>
.content { padding: 20rpx; }   /* rpx = responsive px, 750 = design width */
</style>
```

`view`/`text`/`button` are mini-program-style components, not HTML elements; `uni.*` is the platform API surface; `#ifdef`/`#endif` are conditional-compilation directives resolved per target at build time.

## Architecture / How It Works

uni-app compiles a Vue single-file-component project into each platform's native shape. Two files drive the build: `pages.json` (route table + per-page window config) and `manifest.json` (app id, permissions, per-platform settings). `App.vue` and `main.js` are the entry points.

**Original uni-app** renders differently per target. On mini-programs it transpiles to that platform's WXML/WXSS-style templates and runs in the vendor's logic/render split. On native apps the default renderer is a web-view running the Vue pages, with a separate JS logic layer — architecturally the same "two-thread" model as a mini-program. For performance-sensitive screens it offers **nvue**, a native renderer historically based on Weex, so a single app can mix web-view and native-rendered pages.

**uni-app x** removes the web-view compatibility layer on native. UTS source is compiled ahead of time to the platform's own language (Kotlin/Swift/ArkTS), and `uvue` drives native views directly. This is the strategic bet: instead of shipping a JS engine + web-view and paying the bridge cost, the app becomes native code with a Vue-shaped authoring model.

**Conditional compilation** (`#ifdef MP-WEIXIN`, `#ifndef H5`, etc.) is the core cross-platform mechanism and also the honest admission that "write once, run everywhere" leaks: real apps carry per-platform branches for APIs, payment, share, and login that only exist on one super-app. The abstraction covers the common 80%; the platform-specific 20% is explicit in the source.

The framework is developed as a monorepo. Historically it shipped Vue 2 and Vue 3 lines on separate branches (`uni-app-vue2`, `uni-app-vue3`); uni-app x lives on the `uni-app-x` branch and is where new investment is going.

## Production Notes

**Documentation is Chinese-first.** Official docs, the plugin market, and the community Q&A (ask.dcloud.net.cn) are primarily Chinese. English docs exist but lag and are incomplete. This is the single biggest adoption barrier for non-Chinese teams.

**HBuilderX vs CLI.** Some features and one-click build targets are smoother (or only documented) inside HBuilderX. The CLI (Vite/Vue-CLI) covers most of the surface but you will occasionally hit a workflow the docs assume you're doing in the IDE.

**"Write once" is aspirational.** Budget for conditional-compilation branches on every non-trivial app. Login, payment, sharing, push, and file APIs differ per super-app; the `uni.*` layer normalizes the easy calls but not the platform-specific auth/pay flows.

**uni-app vs uni-app x is a real fork in the road.** uni-app (JS + web-view/Weex) is mature and has the deepest plugin ecosystem. uni-app x (UTS + uvue native) is newer, faster on native, but has a smaller module ecosystem and a moving API surface — not every plugin from the old world has a uni-app x equivalent yet. Choose deliberately; migration between them is not free.

**Mini-program constraints leak upward.** Because the runtime mirrors the WeChat model, you inherit its limits: restricted DOM/BOM, package-size ceilings per mini-program platform, and a logic/render thread split that makes some web patterns (direct DOM access, large synchronous data passing across the bridge) slow or unavailable — even on the web/app targets.

**nvue caveats (original uni-app).** The native (Weex-based) renderer supports a narrower CSS subset than the web-view renderer (flexbox-centric, no cascading in the browser sense). Mixing vue and nvue pages means maintaining two styling mental models.

**Debugging surface is wide.** A bug can live in your Vue code, in the uni-app compiler, in a specific mini-program vendor's runtime, or in native bridge code. Reproducing per-platform is part of the job; a passing H5 build does not guarantee the WeChat or iOS build behaves the same.

## When to Use / When Not

**Use when:**
- Your product must ship as a Chinese mini-program (especially WeChat) and a native app from one codebase.
- Your team already knows Vue and works in Chinese-language docs comfortably.
- You want DCloud's integrated stack (uniCloud, uni-id, uni-push, plugin market) rather than assembling one.
- You need to reach quick-apps / HarmonyOS alongside the majors.

**Avoid when:**
- You have no mini-program requirement and just want a Vue-flavored native app — React Native or Capacitor have larger English ecosystems.
- Your team can't work primarily from Chinese documentation.
- You need a browser-native web app with full DOM access — the mini-program-shaped abstraction gets in the way.
- You want a slow-moving, rarely-breaking framework — the uni-app / uni-app x split and UTS are actively evolving.

## Alternatives

- NervJS/taro — the closest competitor: JD-originated, React or Vue, targets the same mini-programs plus React Native and web. Prefer it when your team is React-first.
- facebook/react-native — use when you want native iOS/Android with the largest ecosystem and no mini-program target.
- flutter/flutter — use when you want a single high-performance native UI toolkit and are willing to leave the JS/Vue and mini-program worlds entirely.
- ionic-team/capacitor — use when your app is essentially a web app you want wrapped as native, without the mini-program dimension.
- Rax / Remax — older React-based mini-program frameworks; mostly relevant for legacy Alibaba-ecosystem projects rather than new builds.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-07 | Repository created; framework announced by DCloud[^1]. |
| 2.x | 2019–2020 | Vue 2 line matures; broad mini-program target coverage. |
| 3.x | 2021 | Vue 3 + Vite support; `uni-app-vue3` branch. |
| uni-app x | 2023–2024 | UTS language + uvue native engine introduced; becomes repo default branch[^2]. |
| — | 2024–2025 | HarmonyOS Next / ArkTS target expands under uni-app x. |

Dates are approximate at year granularity; DCloud versions the toolchain (HBuilderX) more visibly than the framework repo, so exact per-feature release dates are not reliably tagged in this repo.

## References

[^1]: uni-app README and official introduction, DCloud. https://uniapp.dcloud.io
[^2]: uni-app x documentation, DCloud. https://doc.dcloud.net.cn/uni-app-x/

## Tags

vue, cross-platform, mini-program, mobile, ios, android, harmonyos, javascript, typescript, uts, hybrid-app, dcloud
