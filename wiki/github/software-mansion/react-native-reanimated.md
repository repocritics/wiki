# software-mansion/react-native-reanimated

> React Native animation library that runs animation logic on a separate UI-thread JavaScript runtime ("worklets") instead of the bridge.

[GitHub repo](https://github.com/software-mansion/react-native-reanimated) ·
[Official website](https://docs.swmansion.com/react-native-reanimated/) ·
[License: MIT](https://github.com/software-mansion/react-native-reanimated/blob/main/LICENSE)

## Overview

Reanimated is Software Mansion's animation library for React Native, first
released in 2018 as a lower-level, more capable alternative to React Native's
built-in `Animated` API[^1]. The problem it solves is structural: in classic
React Native, JavaScript and the native UI run on separate threads connected by
an asynchronous bridge, so any animation driven from JS stutters whenever the JS
thread is busy. Reanimated moves the animation code off the JS thread entirely.

The defining mechanism, introduced in Reanimated 2 (2021), is the **worklet**: a
JavaScript function tagged with a `"worklet"` directive that a Babel plugin
extracts and runs on a second JS runtime pinned to the UI thread[^2]. Shared
values (`useSharedValue`) live in memory both runtimes can read, so an animation
can update at 60/120 fps without a round-trip to the main JS thread. This was a
large conceptual jump from Reanimated 1's declarative node graph
(`Animated.cond`, `block`, `interpolate` as data structures) — powerful but hard
to read and debug.

The tradeoff is a mental model most React developers do not already have: two
runtimes, values that exist outside React's render cycle, closures captured
by-copy across a thread boundary, and a build step that must be configured
exactly right. Reanimated is the default choice for non-trivial RN animation and
gesture work — it underpins much of the ecosystem, including
`react-native-gesture-handler` and most bottom-sheet/carousel libraries — but it
is not a library you use casually.

## Getting Started

```bash
npm install react-native-reanimated
# or: yarn add react-native-reanimated
```

Add the Babel plugin — it must be listed **last** in `babel.config.js`:

```js
module.exports = {
  presets: ["babel-preset-expo"], // or module:@react-native/babel-preset
  plugins: ["react-native-reanimated/plugin"], // keep last
};
```

```tsx
import Animated, {
  useSharedValue, useAnimatedStyle, withSpring,
} from "react-native-reanimated";
import { Button, View } from "react-native";

export default function Box() {
  const offset = useSharedValue(0);
  const style = useAnimatedStyle(() => ({
    transform: [{ translateX: offset.value }], // runs on the UI thread
  }));
  return (
    <View>
      <Animated.View style={[{ width: 80, height: 80, backgroundColor: "tomato" }, style]} />
      <Button title="Move" onPress={() => (offset.value = withSpring(100))} />
    </View>
  );
}
```

## Architecture / How It Works

The library has three cooperating layers:

1. **Worklets runtime.** A separate JS runtime (Hermes or JSC) executes on the
   UI thread. Worklet functions are serialized — their code and captured
   closure variables are copied — and installed into that runtime. As of
   Reanimated 4 this machinery was extracted into a standalone
   `react-native-worklets` package that Reanimated depends on[^3]; the repo now
   ships both packages from one monorepo.
2. **Babel plugin.** `react-native-reanimated/plugin` scans for the `"worklet"`
   directive (and auto-workletizes the callbacks passed to hooks like
   `useAnimatedStyle`, `useDerivedValue`, gesture handlers). It rewrites those
   functions so their source and closure can cross the runtime boundary. If the
   plugin is missing or misordered, worklets silently fall back to running on
   the JS thread or throw at runtime — a large share of "it doesn't animate"
   bug reports trace to this.
3. **Shared values & native driver.** `useSharedValue` creates a boxed value
   readable from both runtimes; `withTiming`/`withSpring`/`withDecay` mutate it
   frame-by-frame on the UI thread, and `useAnimatedStyle` applies the result to
   a native view directly, bypassing React re-renders.

Reanimated 4 rebased the public animation model on a **CSS-like API** (CSS
animations and transitions expressed in JS style objects) layered on the same
worklets core, and dropped support for React Native's legacy ("Paper")
architecture — 4.x requires the New Architecture (Fabric)[^4]. Apps on the old
architecture must stay on the 3.x line.

## Production Notes

- **Reading `.value` during render is a bug.** Shared values live outside
  React's state. Reading `sharedValue.value` in a component body does not
  subscribe the component and will warn; UI-thread reads belong inside worklets.
- **Closure capture is by copy, not by reference.** A worklet sees a snapshot of
  the variables it captured when it was created. Objects mutated on the JS
  thread afterward are not reflected. Cross-thread mutation goes through shared
  values or `runOnUI` / `runOnJS`.
- **The Babel plugin is the number-one footgun.** It must be last in the plugin
  list, and adding it (or reordering) requires clearing the Metro cache
  (`--reset-cache`). Symptoms of a misconfigured plugin are subtle: animations
  that jump instead of interpolate, or callbacks that never fire.
- **Version coupling is tight.** Reanimated pins to specific React Native ranges
  and interacts with `react-native-gesture-handler`, Expo SDK, and Hermes.
  Upgrading RN often forces a coordinated Reanimated bump; mismatches produce
  native build failures rather than JS errors. On Expo, a given Expo Go binary
  bundles only one Reanimated version, so real use needs a development build —
  the 4.x New-Architecture requirement compounds this.
- **Layout animations and shared-element transitions** are the least stable
  surface — behavior differs across platforms and RN versions, and edge cases
  (unmount timing, lists) are common issue sources. Debugging is also harder:
  worklet stack traces come from the second runtime and remote JS debugging is
  incompatible with the UI-thread runtime, pushing you toward on-device logging.

## When to Use / When Not

**Use when:**
- You need gesture-driven or continuously interactive animation that stays smooth
  while the JS thread is busy.
- You are already on `react-native-gesture-handler` or a library that depends on
  Reanimated (bottom sheets, carousels, drawer navigators).
- You want frame-accurate control beyond what `Animated` / `LayoutAnimation` give.

**Avoid when:**
- A single fade or one-off transition would do — `Animated` or the `Animated` API
  in RN core, or a small CSS-transition wrapper, is less setup and risk.
- You cannot adopt the New Architecture and want the newest (4.x) features.
- You are building for a constrained/managed setup where adding native deps and a
  Babel plugin is not acceptable (some restricted Expo Go workflows).

## Alternatives

- software-mansion/react-native-gesture-handler — companion, not competitor; pair it with Reanimated for gesture-driven motion.
- facebook/react-native — the built-in `Animated` API; use it when animations are simple and you want zero extra deps.
- wcandillon/react-native-redash — helper toolkit built on top of Reanimated; use it to avoid rewriting common worklet math.
- react-native-skia (Shopify) — use instead when you need custom canvas/graphics rendering rather than view-property animation.
- legendapp/legend-motion or moti (nandorojo/moti) — declarative animation wrappers over Reanimated; use when you want a simpler API and accept less control.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019 | Declarative node-graph API (`Animated.cond`/`block`/`interpolate`)[^1]. |
| 2.0 | 2021 | Worklets + shared values; animation runs on the UI thread[^2]. |
| 3.0 | 2023 | Dropped Reanimated 1 API; layout animations, shared-element transitions. |
| 4.0 | 2025 | CSS-based animation API; Worklets split into `react-native-worklets`; New Architecture only[^4]. |

## References

[^1]: Software Mansion, react-native-reanimated repository (created 2018). https://github.com/software-mansion/react-native-reanimated
[^2]: Reanimated worklets documentation. https://docs.swmansion.com/react-native-reanimated/docs/fundamentals/glossary/#worklet
[^3]: React Native Worklets documentation. https://docs.swmansion.com/react-native-worklets/
[^4]: Reanimated documentation — Reanimated 4 and New Architecture requirement. https://docs.swmansion.com/react-native-reanimated/

## Tags

react-native, animation, gestures, worklets, mobile, typescript, javascript, ui-thread, multithreading, expo, software-mansion
