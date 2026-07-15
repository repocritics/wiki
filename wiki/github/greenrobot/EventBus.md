# greenrobot/EventBus

> Annotation-driven publish/subscribe event bus for Android and Java that decouples senders from receivers within a single process.

[GitHub repo](https://github.com/greenrobot/EventBus) ·
[Official website](https://greenrobot.org/eventbus/) ·
[License: Apache-2.0](https://github.com/greenrobot/EventBus/blob/master/LICENSE)

## Overview

EventBus is an in-process publish/subscribe library first released by Markus Junginger (greenrobot) in 2012[^1]. Its purpose is narrow: let one component post an object ("event") and let arbitrary other components receive it by declaring an annotated method, without either side holding a reference to the other. On Android this was historically used to pass data between Activities, Fragments, background threads, and Services without threading callbacks and interfaces through every layer.

The defining tradeoff is decoupling versus traceability. Because subscribers are wired up by annotation at runtime rather than by explicit method calls, the compiler cannot tell you who handles a given event, and "find usages" on an event class does not reveal the delivery graph. This is the same property that makes EventBus convenient for small apps and painful to reason about in large ones. The library is deliberately single-process and in-memory; it is not a message queue, has no persistence, and does not cross process or network boundaries.

As of 2026 EventBus is mature and widely deployed (the project cites apps with over one billion combined installs[^1]) but no longer actively evolving — the last published commit was in early 2024 and current Android practice has largely shifted to lifecycle-aware and reactive primitives (LiveData, Kotlin `Flow`/`SharedFlow`). It remains a reasonable choice for existing codebases and simple decoupling, not a default for new architecture.

## Getting Started

Available on Maven Central under `org.greenrobot`. The artifact split (since 3.2) separates the Android build from the pure-Java build:

```groovy
// Android
implementation("org.greenrobot:eventbus:3.3.1")
// Pure Java / JVM
implementation("org.greenrobot:eventbus-java:3.3.1")
```

The three-step usage: define an event, subscribe, post.

```java
// 1. Event — any plain class
public static class MessageEvent {
    public final String text;
    public MessageEvent(String text) { this.text = text; }
}

// 2. Subscriber — annotated method; register/unregister on lifecycle
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessageEvent(MessageEvent event) {
    textView.setText(event.text);   // runs on the main thread
}

@Override public void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}

@Override public void onStop() {
    super.onStop();
    EventBus.getDefault().unregister(this);   // omitting this leaks
}

// 3. Post from anywhere
EventBus.getDefault().post(new MessageEvent("hello"));
```

## Architecture / How It Works

`EventBus.getDefault()` returns a process-wide singleton; you can also build isolated instances with `EventBus.builder()`. On `register(subscriber)`, EventBus discovers every method annotated `@Subscribe` on the subscriber's class and indexes them by their single parameter type. `post(event)` looks up subscribers whose parameter type is assignable from the event's runtime class and dispatches to each.

Subscriber discovery has two paths. By default it uses **reflection** at registration time to enumerate annotated methods. Because this can be slow and is fragile under aggressive obfuscation/optimization, greenrobot ships an **annotation processor** (`eventbus-annotation-processor`) that generates a compile-time *subscriber index* — a static table of subscriptions — which EventBus consults instead of reflecting. The project explicitly recommends the index to avoid "reflection related problems seen in the wild"[^1].

Dispatch is governed by `ThreadMode` on each subscription:

- `POSTING` — runs synchronously on the posting thread (default; lowest overhead).
- `MAIN` — runs on the Android main thread; posted-from-main calls are synchronous, otherwise queued.
- `MAIN_ORDERED` — always enqueued to the main thread, preserving post order.
- `BACKGROUND` — runs on a single background thread; posts are serialized.
- `ASYNC` — runs on a thread from a pool; concurrent, for long/blocking work.

Additional features layered on top: **sticky events** (`postSticky` retains the last event so late-registering subscribers receive it immediately), **subscriber priorities** with `cancelEventDelivery` to stop propagation, **event inheritance** (a subscriber to a superclass or interface receives all subtype events), and built-in `NoSubscriberEvent` / `SubscriberExceptionEvent` notifications for observability. Event routing is entirely type-based — there are no string topics — but the routing graph is implicit and resolved at runtime.

## Production Notes

**Forgetting to `unregister` leaks the subscriber.** The bus holds a strong reference to every registered object; an Activity or Fragment that registers in `onStart`/`onCreate` and fails to unregister in the matching lifecycle callback is retained past its lifetime. This is the single most common EventBus bug in production Android apps.

**Use the subscriber index in release builds.** Reflection-based registration interacts badly with R8/ProGuard: renamed or stripped methods break discovery, and reflection cost is paid on every `register`. The generated index sidesteps both. The Android artifact ships consumer R8/ProGuard rules[^1], but the index is still the recommended path for non-trivial apps.

**`ThreadMode.ASYNC` is unbounded relative to your post rate.** It draws from a thread pool; a high volume of async-posted events with slow handlers can exhaust threads. `BACKGROUND` serializes onto one thread and can instead build a backlog if a handler blocks. Match the mode to the work.

**Debuggability degrades with scale.** Because delivery is decoupled and type-driven, tracing "why did this run" or "who consumes this event" requires runtime inspection, not static navigation. Event inheritance compounds this: a broad base-type subscriber silently receives every subtype. Large codebases tend to accumulate hard-to-audit event graphs — a frequently cited reason teams migrate away.

**Maintenance status.** The repository has seen no new release since 3.3.1 and its last commit is from early 2024. It is stable and the API is settled, but do not expect fixes, new `ThreadMode`s, or first-class Kotlin coroutine integration. New Android code generally favors `Flow`/`SharedFlow` or `LiveData`, which are lifecycle-aware and statically traceable.

## When to Use / When Not

**Use when:**
- You have an existing project already built around EventBus and it works.
- You need simple one-to-many decoupling within a single JVM/Android process and want minimal boilerplate.
- You want thread-hopping to the main thread from background work without wiring handlers manually.
- Payload is a plain object and you do not need backpressure, persistence, or replay.

**Avoid when:**
- You are starting a new Android app: prefer `StateFlow`/`SharedFlow` or `LiveData` for lifecycle-safety and traceability.
- You need backpressure, cancellation, or stream composition — use a reactive library.
- Event flow is complex enough that implicit runtime routing would obscure control flow.
- You need cross-process or networked messaging — EventBus is in-memory and single-process only.

## Alternatives

- ReactiveX/RxJava — full reactive streams with operators, backpressure, and scheduling; more capable and heavier, steeper learning curve. Use instead when you need stream composition rather than fire-and-forget events.
- Kotlin/kotlinx.coroutines (`SharedFlow`/`StateFlow`) — the idiomatic modern replacement for event/state broadcasting on Android and Kotlin JVM. Use for any new Kotlin codebase.
- google/guava — Guava's `EventBus` offers the same in-process pub/sub pattern for plain Java without an annotation processor. Use when you already depend on Guava and target the JVM, not Android.
- square/otto — the historical peer event bus for Android; deprecated by Square in favor of RxJava. Only relevant to legacy code.
- androidx LiveData — lifecycle-aware observable holder from Jetpack. Use when you want UI state observation tied to Android lifecycles rather than a general bus.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012 | Initial release by greenrobot[^1]. |
| 2.4 | 2015 | Reflection-based subscriber registration era; widely adopted on Android. |
| 3.0 | 2016 | Annotation-based `@Subscribe`, subscriber index via annotation processor, `ThreadMode`s[^2]. |
| 3.2 | 2020 | Split into `eventbus` (Android) and `eventbus-java` (pure Java) artifacts. |
| 3.3.1 | 2023 | Latest release; AndroidX/R8 consumer rules, maintenance-level changes[^2]. |

## References

[^1]: EventBus README and project homepage — greenrobot. https://greenrobot.org/eventbus/
[^2]: EventBus releases and changelog. https://github.com/greenrobot/EventBus/releases

## Tags

java, android, event-bus, publish-subscribe, pub-sub, messaging, decoupling, annotation-processor, jvm, observer-pattern
