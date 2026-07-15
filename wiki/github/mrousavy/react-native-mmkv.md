# mrousavy/react-native-mmkv

> Synchronous key/value storage for React Native, backed by Tencent's C++ MMKV via JSI — roughly 30x faster than AsyncStorage.

[GitHub repo](https://github.com/mrousavy/react-native-mmkv) ·
[Author site](https://mrousavy.com) ·
[License: MIT](https://github.com/mrousavy/react-native-mmkv/blob/main/LICENSE)

## Overview

react-native-mmkv is a thin React Native binding over [Tencent/MMKV](https://github.com/Tencent/MMKV), the memory-mapped key/value store WeChat built for its own mobile clients. The library itself is small; almost all of the storage behavior — mmap-backed writes, append-only serialization, CRC integrity, optional AES encryption — lives in Tencent's C++ core. mrousavy's contribution is the JS-to-native binding that makes that core usable from React Native without touching the async bridge.

The defining property is **synchronous access**. `storage.getString('key')` returns a value on the calling thread with no Promise, no `await`, and no serialization round-trip across the old bridge. This is possible because the library uses JSI (JavaScript Interface) to call directly into C++. The tradeoff is the mirror image of AsyncStorage's: you get microsecond reads and simpler code, but a blocking call on the JS thread and a hard dependency on the New Architecture. It is one of the most widely adopted "JSI-era" native modules and a common backing store for `zustand`, `redux-persist`, `jotai`, and `react-query` persistence.

As of 2026 the current line is **V4**, which is built on [Nitro Modules](https://nitro.margelo.com) (mrousavy's codegen layer, distinct from React Native's own TurboModules) and requires React Native 0.76 or newer[^1]. V3 remains documented separately for apps that cannot yet move to the New Architecture.

## Getting Started

```sh
npm install react-native-mmkv react-native-nitro-modules
cd ios && pod install
```

Expo (managed) needs a prebuild because this is a native module:

```sh
npx expo install react-native-mmkv react-native-nitro-modules
npx expo prebuild
```

```ts
import { createMMKV } from 'react-native-mmkv'

// Create ONE instance and reuse it app-wide.
export const storage = createMMKV()

storage.set('user.name', 'Marc')
storage.set('user.age', 21)
storage.set('loggedIn', true)

const name = storage.getString('user.name')  // 'Marc'
const age = storage.getNumber('user.age')     // 21
```

React hooks re-render on change:

```ts
const [name, setName] = useMMKVString('user.name')
```

## Architecture / How It Works

The stack is three layers: the TypeScript API you import, a generated Nitro binding, and Tencent's C++ MMKV library compiled into your app.

- **JSI, not the bridge.** Calls are synchronous C++ invocations from the JS runtime. There is no message queue, no JSON encode/decode, and no thread hop for a `get`. This is why remote debugging (Chrome/Hermes-over-websocket) does not work — the JS is no longer running in a context that can reach a native C++ pointer[^2]. Use Flipper or React DevTools instead.
- **Nitro Modules (V4).** V4 replaced the hand-written JSI bindings with Nitro-generated ones. Nitro is mrousavy's own codegen (used by react-native-vision-camera and others), not Meta's TurboModules, though both target the New Architecture. This is why `react-native-nitro-modules` is a required peer dependency in V4.
- **The storage engine is Tencent's.** MMKV mmaps a file and appends serialized key/value writes to it; reads are served from the memory-mapped region. Because writes are append-only, the file grows until `trim()` compacts it. Encryption (AES-128 default, AES-256 opt-in) and multi-process mode are Tencent features surfaced through the config object, not reimplementations.
- **Instances and IDs.** Each `createMMKV({ id })` maps to a separate on-disk file under `$(Documents)/mmkv/`. A common pattern is a global instance plus a per-user instance (`user-${id}-storage`) so logout can `deleteMMKV` one file without touching the rest.
- **Web.** A `localStorage`-backed shim exists; if `localStorage` is disabled it degrades to non-persistent in-memory storage — data is silently lost on refresh, which is a real footgun for web builds.

The coupling story is honest and unusually clean: the value comes from Tencent's engine, and the library's own surface area (and its bug surface) is mostly the binding layer.

## Production Notes

- **Values are strings/numbers/booleans/ArrayBuffers — not objects.** There is no built-in object serialization. You `JSON.stringify` on write and `JSON.parse` on read. Large objects mean large synchronous string work on the JS thread.
- **Synchronous means blocking.** Reads are fast enough that this rarely matters, but storing megabyte-scale blobs and reading them on every render will jank the UI thread. MMKV is tuned for many small keys, not for being a document store.
- **File grows until trimmed.** The append-only format means repeated writes to the same key inflate the file. Check `storage.byteSize` and call `storage.trim()` to compact and release memory cache; there is no automatic GC.
- **`compareBeforeSet` is off by default.** If your code repeatedly writes identical values, enabling it avoids redundant disk writes — otherwise every `set` hits disk even when the value is unchanged.
- **Encryption is at-rest, key management is yours.** Passing `encryptionKey` encrypts the file, but the key still has to live somewhere; hardcoding it (as the docs' `'hunter2'` example does) defeats the purpose. Pair it with Keychain/Keystore. `encrypt()`/`decrypt()` can also rotate encryption on an existing instance.
- **New Architecture is mandatory (V4).** V4 requires RN 0.76+ and the New Architecture; there is no bridge fallback. Apps still on old-architecture RN must stay on V3, whose API differs (`new MMKV()` vs `createMMKV()`).
- **Testing is handled.** Jest and Vitest automatically get a mocked in-memory instance, so `createMMKV()` works in unit tests without native code.
- **Single-maintainer, funded by sponsorship.** The README states it is maintained in the author's free time and offers paid enterprise support. Treat it as high-quality but bus-factor-one; the underlying Tencent engine is the more heavily resourced dependency.

## When to Use / When Not

**Use when:**
- You want synchronous, low-latency reads for app state, feature flags, tokens, or cache.
- You are migrating off AsyncStorage and want a near drop-in with far better performance.
- You need a persistence backend for `zustand`/`redux-persist`/`jotai` on device.
- You want at-rest encryption or per-user data isolation via multiple instances.

**Avoid when:**
- You are on old-architecture React Native and cannot adopt the New Architecture (V4 won't run; V3 is your ceiling).
- You need queryable, relational, or large-document storage — reach for SQLite (op-sqlite, WatermelonDB) instead.
- Your data is large binary blobs read on the render path — the synchronous model works against you.
- You are targeting web as a first-class persistent surface — the fallback can silently drop data.

## Alternatives

- Tencent/MMKV — the underlying C++ engine itself; use directly when you are native (iOS/Android) and not in React Native.
- ammarahm-ed/react-native-mmkv-storage — an alternative MMKV binding with a different (async-capable, index-based) API; use when you want its feature set or are already invested in it.
- react-native-async-storage/async-storage — the asynchronous baseline; use when you need broad compatibility and don't care about the perf gap.
- op-engineering/op-sqlite — JSI SQLite; use when you need real queries, relations, or large datasets rather than flat keys.
- margelo/react-native-quick-crypto / Keychain — pair with, not replace: use for the key material behind MMKV encryption.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2021-03 | Initial release; JSI bindings over Tencent MMKV, synchronous API[^3]. |
| 2.0 | 2022 | API and instance-configuration changes; hooks matured. |
| 3.0 | 2024 | New Architecture / TurboModule-era release; V3 docs retained for legacy apps. |
| 4.0 | 2025 | Rebuilt on Nitro Modules; requires RN 0.76+, adds `react-native-nitro-modules` peer dep, `createMMKV()` factory API[^1]. |

## References

[^1]: react-native-mmkv README, "V4 Upgrade Guide" and Limitations (requires React Native 0.76+, Nitro Modules). https://github.com/mrousavy/react-native-mmkv
[^2]: react-native-mmkv README, Limitations — JSI synchronous invocation disables remote (Chrome) debugging. https://github.com/mrousavy/react-native-mmkv#limitations
[^3]: Tencent/MMKV — the underlying memory-mapped key/value engine. https://github.com/Tencent/MMKV

## Tags

react-native, mobile, storage, key-value, jsi, cpp, ios, android, mmkv, native-module, encryption, typescript
