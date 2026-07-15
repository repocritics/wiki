# zxing/zxing

> The original Java barcode image-processing library — 1D/2D encode and decode, now in maintenance-only mode.

[GitHub repo](https://github.com/zxing/zxing) ·
[Documentation](https://zxing.github.io/zxing/) ·
[License: Apache-2.0](https://github.com/zxing/zxing/blob/master/LICENSE)

## Overview

ZXing ("zebra crossing") is a multi-format 1D/2D barcode image-processing library written in pure Java[^1]. It both decodes (reads barcodes from images) and encodes (generates them), covering QR Code, Data Matrix, Aztec, PDF 417, MaxiCode on the 2D side, and UPC-A/E, EAN-8/13, Code 39/93/128, Codabar, ITF on the 1D side. It began as a Google project circa 2008, is best known for the once-ubiquitous Android "Barcode Scanner" app, and remains the upstream reference that a large family of ports (C++, .NET, JavaScript, Python, Dart, Rust) trace back to.

The single most important fact for anyone evaluating it in 2026: **the project is in maintenance mode only**[^2]. The README states there is "no active development or roadmap" and that changes are driven solely by contributed patches for bug fixes and minor enhancements — the maintainers describe it as "DIY". The historically famous Barcode Scanner Android app can no longer be published, does not work on Android 14, and will not be updated; the maintainers explicitly ask people not to file issues about it. The open-issue count sits near zero not because the code is defect-free but because issues are closed aggressively.

ZXing is a pure image-processing library, not a scanner. It has no camera, no frame-capture loop, and no UI. You hand it a grid of luminance values and it returns decoded text and the geometry of the symbol it found. Everything about live scanning (camera access, preview, autofocus, frame throttling) is the caller's responsibility — which is why most real Android usage today goes through the third-party `journeyapps/zxing-android-embedded` wrapper rather than this repository directly.

## Getting Started

The core decoder and the JavaSE image helpers are separate Maven artifacts under `com.google.zxing`:

```xml
<dependency>
  <groupId>com.google.zxing</groupId>
  <artifactId>core</artifactId>
  <version>3.5.3</version>
</dependency>
<dependency>
  <groupId>com.google.zxing</groupId>
  <artifactId>javase</artifactId>
  <version>3.5.3</version>
</dependency>
```

Decode a QR code from a PNG:

```java
import com.google.zxing.*;
import com.google.zxing.client.j2se.BufferedImageLuminanceSource;
import com.google.zxing.common.HybridBinarizer;
import java.awt.image.BufferedImage;
import java.io.File;
import javax.imageio.ImageIO;

BufferedImage image = ImageIO.read(new File("qr.png"));
LuminanceSource source = new BufferedImageLuminanceSource(image);
BinaryBitmap bitmap = new BinaryBitmap(new HybridBinarizer(source));
Result result = new MultiFormatReader().decode(bitmap);
System.out.println(result.getText());
```

Encode one:

```java
BitMatrix matrix = new MultiFormatWriter()
    .encode("https://example.com", BarcodeFormat.QR_CODE, 300, 300);
MatrixToImageWriter.writeToPath(matrix, "PNG", Paths.get("qr.png"));
```

## Architecture / How It Works

Decoding is a fixed pipeline of small interfaces, each replaceable:

1. **`LuminanceSource`** — abstracts a source of grayscale pixels. `BufferedImageLuminanceSource` (JavaSE) and `PlanarYUVLuminanceSource` (Android camera YUV frames) are the common concrete implementations. Cropping and rotation happen here.
2. **`Binarizer`** — converts luminance to black/white. `HybridBinarizer` is the default for 2D and does local thresholding (better under uneven lighting); `GlobalHistogramBinarizer` is faster but weaker on gradients.
3. **`BinaryBitmap`** — the binarized image handed to a reader.
4. **`Reader`** — `MultiFormatReader` inspects hints and delegates to per-format readers (`QRCodeReader`, `DataMatrixReader`, `Code128Reader`, …). Each 2D reader splits into a **detector** (locate finder/alignment patterns, correct perspective) and a **decoder** (extract bits, run Reed–Solomon error correction, apply the format's data mask). Results carry the text, raw bytes, format, and `ResultPoint[]` marking where the symbol was found.

Decoding behavior is tuned through `DecodeHintType`: `TRY_HARDER` (spend more time, higher recall, notably slower), `POSSIBLE_FORMATS` (restrict to a format set for speed and fewer false positives), `PURE_BARCODE` (skip detection when the image is a clean crop), and `CHARACTER_SET`. Encoding mirrors this with `EncodeHintType` (`ERROR_CORRECTION`, `MARGIN`, `CHARACTER_SET`).

The Reed–Solomon codec, `BitMatrix`, and geometry helpers in `core` are the genuinely reusable heart of the library; the format readers are largely independent of each other, which is why downstream ports could translate them piecemeal.

## Production Notes

**Readers are stateful and not thread-safe.** `MultiFormatReader` and the per-format readers keep internal state across a decode. Do not share one instance across threads. Create a reader per thread (or per request) or synchronize access; call `reset()` between decodes when reusing one on a single thread.

**Recognition is classical CV, not ML.** ZXing does a single-pass geometric decode. It performs well on clean, well-lit, roughly frontal symbols and degrades sharply on damaged, low-contrast, motion-blurred, curved, or steeply skewed codes. `TRY_HARDER` helps recall at a real latency cost but does not close the gap with modern detectors. On mobile, Google's ML Kit and the active `zxing-cpp` fork both outperform this library on difficult frames.

**Live scanning is on you.** There is no camera pipeline here. On Android you must convert camera frames (typically YUV_420) to a `PlanarYUVLuminanceSource`, throttle frames yourself, and manage focus. This is exactly the boilerplate `journeyapps/zxing-android-embedded` exists to absorb.

**The Barcode Scanner integration path is effectively dead.** The classic `IntentIntegrator` / `com.google.zxing.client.android.SCAN` intent depends on the Barcode Scanner app being installed — an app that can no longer be published and is broken on Android 14. New Android projects should not build on it.

**Java version and footprint.** `core` is deliberately dependency-free and small, which is a real advantage for server-side batch decoding. Confirm the JDK baseline of the exact release you pin (`core` has stayed conservative to keep the Android port viable) rather than assuming the latest LTS.

**Do not expect fixes.** Because the project is maintenance-only, a bug you hit may sit open or get closed as out-of-scope. Budget for either patching locally or migrating to an actively developed port. This is the dominant long-term risk of adopting it new.

## When to Use / When Not

**Use when:**
- You are on the JVM and need dependency-light server-side barcode encode/decode of clean images (labels, generated codes, document pipelines).
- You want the canonical, license-clean (Apache-2.0) reference implementation and are comfortable owning any fixes.
- You need broad format coverage in one library without native dependencies.

**Avoid when:**
- You are scanning live camera frames on mobile — use ML Kit or `zxing-android-embedded` instead of wiring `core` directly.
- You need best-in-class recognition on hard/damaged codes — `zxing-cpp` or ML-based scanners win.
- You need ongoing upstream maintenance, new formats, or performance work — there is none here.

## Alternatives

- zxing-cpp/zxing-cpp — actively maintained C++ rewrite descended from ZXing, with Android/C/iOS/.NET/Rust/Python/WASM bindings; use when you need native performance or non-JVM targets.
- googlesamples/mlkit (ML Kit Barcode Scanning) — on-device ML scanner; use for high-accuracy live scanning on Android/iOS.
- journeyapps/zxing-android-embedded — wraps ZXing `core` in an embeddable Android camera scanner; use when you specifically want ZXing decoding with a ready-made Android UI.
- micjahn/ZXing.Net — the maintained .NET/C# port; use on the .NET platform.
- mchehab/zbar — long-standing C barcode library (strong on 1D/EAN); use for lightweight native 1D decoding.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | ~2008 | Started as a Google project, originally hosted on Google Code[^1]. |
| 2.x | ~2012 | Widely deployed era; Barcode Scanner Android app at peak adoption. |
| 3.0.0 | ~2014 | Major API cleanup; codebase migrated toward GitHub. |
| 3.3.0 | ~2017 | Minimum Java baseline raised; core kept dependency-free. |
| 3.4.0 | ~2020 | Maintenance-era release; format fixes and hardening. |
| 3.5.0 | ~2022 | Latest minor line; subsequent 3.5.x are patch releases. |
| — | 2024+ | Declared maintenance-only; Barcode Scanner app unpublishable, broken on Android 14[^2]. |

## References

[^1]: ZXing README and project wiki — "ZXing ('zebra crossing') is an open-source, multi-format 1D/2D barcode image processing library implemented in Java." https://github.com/zxing/zxing
[^2]: ZXing README, "Project in Maintenance Mode Only" — states no active development or roadmap, and that the Barcode Scanner app can no longer be published and does not work with Android 14. https://github.com/zxing/zxing/blob/master/README.md

## Tags

java, barcode, qr-code, qr, datamatrix, image-processing, android, computer-vision, decoder, encoder, maintenance-mode
