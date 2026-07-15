# AvaloniaUI/Avalonia

> Cross-platform .NET UI framework that renders its own controls with Skia — XAML and MVVM across desktop, mobile, and WebAssembly.

[GitHub repo](https://github.com/AvaloniaUI/Avalonia) ·
[Official website](https://avaloniaui.net) ·
[License: MIT](https://github.com/AvaloniaUI/Avalonia/blob/master/licence.md)

## Overview

Avalonia is a UI framework for .NET that targets Windows, macOS, Linux, iOS, Android, and WebAssembly from a single codebase using XAML and C#[^1]. It is widely described as the spiritual successor to WPF: XAML developers get a familiar markup dialect, `DataContext`-based binding, and an MVVM-first model, but the APIs are a rewrite rather than a 1:1 port. The project was started in 2013 under the name "Perspex" and later renamed Avalonia.

The defining architectural decision is that Avalonia **does not use native platform controls**. Every widget — buttons, text boxes, list boxes — is drawn by Avalonia itself, by default through Skia (via SkiaSharp). This is the same philosophy as Flutter or WPF and the opposite of .NET MAUI or Uno's native mode. The payoff is pixel-identical rendering and behavior across every platform, plus full control over styling. The cost is that the UI does not automatically inherit native look, feel, accessibility, or platform text/IME behavior — Avalonia reimplements those, and coverage varies by platform.

Avalonia is backed by a commercial company (AvaloniaUI) that funds core development and sells adjacent products, notably **Avalonia XPF**, a per-app, per-platform commercial license that runs existing WPF applications on macOS and Linux with minimal code changes[^2]. The open-source framework is MIT-licensed; XPF is not. Production adopters cited by the project include JetBrains (Rider's UI), Unity, and GitHub[^1].

## Getting Started

Install the templates and scaffold an app with the .NET CLI:

```bash
dotnet new install Avalonia.Templates
dotnet new avalonia.mvvm -o MyApp
cd MyApp
dotnet run
```

A minimal view in XAML with a compiled binding:

```xml
<!-- MainWindow.axaml -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyApp.MainWindow"
        x:DataType="vm:MainViewModel"
        Title="MyApp">
  <StackPanel Margin="16" Spacing="8">
    <TextBox Text="{Binding Name}" Watermark="Your name"/>
    <TextBlock Text="{Binding Greeting}"/>
    <Button Content="Greet" Command="{Binding GreetCommand}"/>
  </StackPanel>
</Window>
```

Selectors style controls with a CSS-like syntax rather than WPF's implicit-key resource model:

```xml
<Style Selector="Button.primary:pointerover">
  <Setter Property="Background" Value="#2563EB"/>
</Style>
```

## Architecture / How It Works

**Rendering.** Avalonia builds a visual tree and rasterizes it through a rendering backend — Skia by default, with a legacy Direct2D path historically available on Windows. On the browser target, output goes to an HTML canvas driven by the .NET WebAssembly runtime. Since Avalonia 11 (2023) rendering runs through a **compositor** with a separate render thread, modeled loosely on WinUI's composition API; this decoupled the UI thread from raster work and enabled smoother animation and partial invalidation.

**Binding and MVVM.** The framework supports classic reflection-based `{Binding}` and, more importantly, **compiled bindings** (`x:CompileBindings` / `x:DataType`), which resolve binding paths at build time for type safety and speed — a meaningful improvement over WPF's runtime reflection. MVVM is the assumed pattern; ReactiveUI has first-class integration and CommunityToolkit.Mvvm is a common lightweight alternative.

**Styling.** Styles use selectors (`Button.primary:pointerover`) instead of WPF's implicit `Style`/`ControlTemplate` keying. `ControlTheme` packages a control's default template and setters. Two themes ship in-box: Fluent (the default) and Simple. Control templating is fully open — any control can be re-skinned in XAML.

**Platform layer.** A thin platform abstraction handles windowing, input, clipboard, and rendering surface per OS (Win32, macOS/AppKit, X11/Wayland, UIKit, Android, Browser). Application code is platform-agnostic; only the head projects (`.Desktop`, `.Android`, `.iOS`, `.Browser`) differ. Everything ships as NuGet packages (`Avalonia`, `Avalonia.Desktop`, `Avalonia.Themes.Fluent`, etc.).

The self-rendering model is the source of both the framework's consistency and its recurring caveats: anything a native toolkit gives for free (accessibility trees, native text shaping, IME, high-DPI quirks, right-click context menus) has to be implemented inside Avalonia, and desktop is more mature than mobile or browser on these fronts.

## Production Notes

- **App size.** Bundling the .NET runtime plus Skia produces large binaries. Desktop self-contained builds are tens of MB; WebAssembly apps download a multi-MB runtime on first load. Trimming, `Avalonia.Browser` AOT, and Blazor-style caching mitigate but do not eliminate this.
- **Trimming / NativeAOT.** Support has improved, but reflection-heavy patterns (some ReactiveUI usage, non-compiled bindings, dynamic XAML) can break under aggressive trimming or AOT. Prefer compiled bindings and test trimmed builds early — trimming failures often surface only at runtime as missing types.
- **Accessibility varies by platform.** Automation/accessibility is wired up (UIA on Windows, and macOS support), but because controls are custom-drawn, screen-reader coverage is not automatic and lags native toolkits in places. Audit if accessibility is a hard requirement.
- **Mobile and browser are younger.** Desktop (Windows/macOS/Linux) is the battle-tested core. iOS, Android, and WASM are supported and shipping, but expect more rough edges, larger performance gaps, and thinner community answers than desktop.
- **Text and fonts.** Custom text stack means font fallback, complex-script shaping, and IME behavior can differ from native apps. Bundle fonts you depend on rather than assuming platform availability.
- **Upgrade friction.** The jump from the 0.10 line to 11.0 was a large rewrite (new compositor, binding, and styling changes). The 11 → 12 transition again ships documented breaking changes[^3]. Pin versions and read the breaking-change notes before major upgrades.
- **WPF migration is not free.** Avalonia is WPF-like, not WPF-compatible. Porting a real WPF app means rewriting styles/templates and adjusting APIs; the drop-in path is the commercial XPF product, not open-source Avalonia.

## When to Use / When Not

**Use when:**
- You want one .NET/XAML codebase to render identically on Windows, macOS, Linux, and optionally mobile/WASM.
- You're a WPF team wanting a modern, cross-platform, actively developed successor.
- Pixel-consistent, fully themeable UI matters more than native platform look-and-feel.
- You're building developer tools, desktop utilities, or dashboards (its strongest niche).

**Avoid when:**
- You need genuine native controls and platform-native feel per OS — use .NET MAUI or Uno's native mode.
- Accessibility must be first-class and native out of the box.
- Mobile is your primary target with heavy platform-specific integration needs.
- You want a zero-cost WPF-to-macOS/Linux port (that path is commercial XPF).
- Minimal binary/download size is critical (Skia + .NET runtime is heavy).

## Alternatives

- dotnet/maui — Microsoft's official cross-platform .NET UI; renders with native controls, mobile-first. Use when native look-and-feel and first-party support outweigh cross-platform pixel consistency.
- unoplatform/uno — mirrors the WinUI/WinRT API across platforms, can render native or via Skia. Use when you want WinUI API compatibility or a native-control option.
- dotnet/wpf — Windows-only, mature, the model Avalonia emulates. Use when you ship only to Windows and want the reference toolkit.
- flutter/flutter — same self-rendering philosophy, non-.NET (Dart). Use when you're outside the .NET ecosystem or want the largest mobile-first community.
- electron/electron — web tech for desktop. Use when your team is web-first and .NET is not a constraint.

## History

| Version | Date | Notes |
|---------|------|-------|
| Perspex (repo created) | 2013-12 | Project started by Steven Kirk; later renamed Avalonia. |
| 0.10 | 2021 | Long-lived final 0.x line; last release before the version jump. |
| 11.0 | 2023-07 | Major rewrite: compositor/render-thread pipeline, styling and binding changes; versioning jumped from 0.10 to 11[^3]. |
| 11.x | 2023–2025 | Point releases hardening desktop, mobile, and browser targets. |
| 12.0 | in development | Documented breaking changes vs 11[^3]. |

## References

[^1]: Avalonia README and project site — cross-platform scope and production adopters. https://avaloniaui.net
[^2]: Avalonia XPF — commercial cross-platform WPF product. https://avaloniaui.net/xpf
[^3]: Avalonia documentation, including Avalonia 11→12 breaking changes. https://docs.avaloniaui.net

## Tags

csharp, dotnet, xaml, cross-platform, desktop-gui, mvvm, ui-framework, wpf-successor, skia, mobile, webassembly
