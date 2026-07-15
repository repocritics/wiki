# robolectric/robolectric

> Runs Android unit tests inside a plain JVM by simulating the framework with "shadow" objects — no emulator, no device.

[GitHub repo](https://github.com/robolectric/robolectric) ·
[Official website](https://robolectric.org) ·
[License: MIT](https://github.com/robolectric/robolectric/blob/master/LICENSE)

## Overview

Robolectric lets you run Android tests on the JVM instead of on an emulator or device. The problem it solves is old and specific: the `android.jar` you compile against ships stub methods that throw `RuntimeException("Stub!")` at runtime, so you cannot execute framework code in a normal JUnit process. Robolectric replaces those stubs at class-load time with working implementations, so an `Activity`, `View`, `SharedPreferences`, or `Looper` behaves close enough to the real thing for a unit test to exercise it. Its own README puts the payoff at roughly 10x faster than cold-started emulator tests[^1].

The project started at Pivotal Labs (the LICENSE still credits Xtreme Labs, Pivotal Labs, and Google)[^2] and is now developed primarily by Google, which maintains an internal tree synced to the public repo via Copybara[^3]. It has been the standard answer to "how do I unit-test Android code that touches the framework" for over a decade — but it is a simulation, and that is its defining tension: shadows approximate device behavior, and a passing Robolectric test is not proof that code works on a real device. Teams use it as a fast deterministic coverage layer and keep a smaller layer of instrumented tests for what simulation gets wrong. As of 2026 it carries ~6.0k stars and ~1.4k forks, ships releases every few months (latest 4.16.1), and supports 14 Android versions from M (API 23) to Baklava (API 36)[^1].

## Getting Started

```groovy
// module build.gradle — Robolectric is a test-only dependency
testImplementation("junit:junit:4.13.2")
testImplementation("org.robolectric:robolectric:4.16.1")
testImplementation("androidx.test.ext:junit:1.3.0")

// enable resource + asset access from the merged manifest
android { testOptions { unitTests { isIncludeAndroidResources = true } } }
```

```java
@RunWith(AndroidJUnit4.class)   // routes to the Robolectric test runner
public class MyActivityTest {
  @Test
  public void clickingButton_changesMessage() {
    try (ActivityController<MyActivity> controller =
             Robolectric.buildActivity(MyActivity.class)) {
      controller.setup();                       // drive lifecycle to RESUMED
      MyActivity activity = controller.get();

      activity.findViewById(R.id.button).performClick();
      TextView text = activity.findViewById(R.id.text);
      assertEquals("Robolectric Rocks!", text.getText().toString());
    }
  }
}
```

## Architecture / How It Works

The core mechanism is a **custom class loader plus bytecode instrumentation**. When a Robolectric test runs, framework classes are loaded through a sandbox class loader (backed by ASM) that rewrites them so method bodies can be intercepted. Interception is directed at **shadow classes** — plain Java classes annotated `@Implements(View.class)` whose `@Implementation` methods stand in for (or supplement) the real ones. `Shadows.shadowOf(view)` hands you the shadow to inspect or drive simulated state.

Robolectric does not hand-write the whole framework. It downloads prebuilt **`android-all` jars** — the actual AOSP framework classes for each SDK level — from Maven, instruments them, and runs your code against real framework logic wherever possible; shadows fill the gaps, chiefly native methods that have no JVM implementation. This is the 4.x model: earlier versions leaned far more heavily on shadows re-implementing behavior, which drifted from AOSP, so running real framework code narrowed that gap considerably. For native-backed graphics (`Bitmap`, `Canvas`, `Path`), recent versions can load real Android native libraries instead of returning stub values, improving fidelity at the cost of platform-specific native binaries.

Two pieces of the runtime deserve special mention because they show up in most test failures:

- **`@Config`** — annotation to set the simulated SDK level, application class, and resource qualifiers per test or per class. Changing SDK level is not free (see below).
- **`ShadowLooper`** — Android's message queue is simulated. Since 4.4 the default looper mode is `PAUSED`: tasks posted to the main `Handler` do **not** run until you advance the scheduler (`shadowOf(getMainLooper()).idle()`). Forgetting this is the single most common source of "my callback never fired" confusion. The older `LEGACY` mode auto-ran tasks and behaved differently.

Each simulated SDK level gets its own sandbox class loader, which is what makes a test reproducible but also drives the framework's biggest performance characteristics.

## Production Notes

**First run is slow and network-bound.** The `android-all` instrumented jars are large (tens of MB per SDK) and are fetched from Maven on first use. CI without a warm dependency cache pays this every build; teams pin and pre-warm these artifacts.

**Every distinct SDK level is a separate class loader.** A suite that runs the same tests across API 23/28/34 loads and instruments the framework three times. This multiplies both wall-clock time and heap; parameterizing across many SDKs is a real cost, not a free matrix.

**It is a simulation, and shadows can lie.** A shadow may return a plausible default that a real device would compute differently, or a framework path may be unimplemented. Passing Robolectric tests do not replace instrumented tests for anything sensitive to real rendering, real hardware, or exact platform behavior. The correct posture is a large fast Robolectric layer plus a smaller Espresso / instrumented layer.

**Native graphics is capable but platform-touchy.** Enabling real native graphics pulls in native libraries; historically Apple Silicon and some CI images have needed specific versions or fallbacks to avoid link/crash errors. Confirm your runner architecture is supported before depending on pixel-level `Bitmap` behavior.

**Version coupling.** The Robolectric version, the Android Gradle Plugin, and the `compileSdk` you test against are coupled; a release supports a bounded range of API levels, so upgrading `compileSdk` often forces a Robolectric upgrade and vice versa. Resource access also depends on `includeAndroidResources` and on AGP producing the merged manifest.

**Looper and threading.** With the main looper paused by default, `postDelayed`, main-thread coroutine/RxJava dispatchers, and `AsyncTask`-style work require explicit scheduler advancement. Tests that "work sometimes" are usually missing an `idle()`.

## When to Use / When Not

**Use when:**
- You need fast, deterministic unit tests for code that touches the Android framework (lifecycle, `SharedPreferences`, `Parcelable`, resources, `Handler`).
- You want to run those tests in a normal JVM/CI process without an emulator.
- You want a broad, cheap coverage layer that fails in seconds, not minutes.

**Avoid when:**
- You need proof of real on-device behavior — rendering, gestures, real GPU, vendor differences. Use instrumented tests.
- Your code has little framework contact; plain JUnit plus a mocking library is lighter and faster.
- You need pixel-accurate UI snapshots — a LayoutLib-based renderer fits better than shadow-based simulation.

## Alternatives

- androidx.test / Espresso (androidx test libraries) — use when you need real instrumented behavior on a device or emulator, not a JVM simulation.
- cashapp/paparazzi — use when you want to render and screenshot Android UI on the JVM via LayoutLib instead of simulating framework behavior with shadows.
- mockito/mockito — use when your class touches only a few Android types you can stub directly, so you do not need a simulated framework at all.
- mockk/mockk — use for Kotlin-first mocking of Android dependencies in pure unit tests with no Android runtime.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2010-08-28 | Origin at Pivotal / Xtreme Labs[^2]. |
| 3.0 | 2015 | Major resource-loading and runtime rework. |
| 4.0 | 2018-10 | Runs instrumented real AOSP framework jars; AndroidX test integration; reduced legacy shadow reliance. |
| 4.4 | 2020 | `PAUSED` looper becomes the default scheduler mode. |
| 4.10 | 2023 | Native graphics via real Android native libraries. |
| 4.16.1 | 2026 | Current release; supports API 23–36 (M through Baklava)[^1]. |

## References

[^1]: Robolectric README — "industry-standard unit testing framework for Android… run 10x faster… 14 different versions of Android, ranging from M (API 23) to Baklava (API 36)." https://github.com/robolectric/robolectric
[^2]: Robolectric LICENSE — MIT, "Copyright (c) 2010 Xtreme Labs, Pivotal Labs and Google Inc." https://github.com/robolectric/robolectric/blob/master/LICENSE
[^3]: Robolectric README, "Development model" — source-of-truth GitHub repo plus an internal Google tree synced to the `google` branch via Copybara. https://github.com/robolectric/robolectric
[^4]: Robolectric architecture overview. https://robolectric.org/architecture

## Tags

java, android, unit-testing, jvm, testing, shadows, bytecode-instrumentation, android-framework, test-framework, junit
