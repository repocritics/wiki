# MonoGame/MonoGame

> An open-source, code-first re-implementation of Microsoft's discontinued XNA 4.0 framework for building cross-platform C# games.

[GitHub repo](https://github.com/MonoGame/MonoGame) ·
[Official website](https://monogame.net) ·
[License: MS-PL](https://github.com/MonoGame/MonoGame/blob/develop/LICENSE.txt)

## Overview

MonoGame is a low-level .NET game framework, not a game engine. It descends from XNA 4.0, the API Microsoft shipped for Xbox 360 and Windows and then abandoned in 2013[^1]. When XNA was killed, MonoGame continued the API surface almost verbatim on top of a portable rendering and input layer, which is why XNA-era books, samples, and tutorials still largely apply. The project has shipped commercially proven titles including Stardew Valley, Celeste, Streets of Rage 4, and Carrion[^2].

The defining tension is that MonoGame gives you the XNA object model — `Game`, `GraphicsDevice`, `SpriteBatch`, `ContentManager`, a fixed/variable timestep loop — and almost nothing above it. There is no scene graph, no entity-component system, no editor, no physics, no UI toolkit. You write the game loop, load your own assets through the content pipeline, and draw everything yourself. This is a deliberate floor-not-ceiling design: teams who want total control over the frame and a small dependency surface choose it; teams who want an editor and batteries-included tooling generally do not.

MonoGame is community-governed and donation-funded, run by a small core team rather than a corporation[^3]. Development happens on the `develop` branch; releases are distributed as NuGet packages and `dotnet new` templates.

## Getting Started

```bash
# Install the project + item templates (once)
dotnet new install MonoGame.Templates.CSharp

# Create and run a cross-platform desktop game
dotnet new mgdesktopgl -o MyGame
cd MyGame
dotnet run
```

```csharp
// Game1.cs — the XNA-shaped loop MonoGame inherits
public class Game1 : Game
{
    private GraphicsDeviceManager _graphics;
    private SpriteBatch _spriteBatch;

    public Game1()
    {
        _graphics = new GraphicsDeviceManager(this);
        Content.RootDirectory = "Content";
    }

    protected override void LoadContent() =>
        _spriteBatch = new SpriteBatch(GraphicsDevice);

    protected override void Update(GameTime gameTime)
    {
        // your simulation step
        base.Update(gameTime);
    }

    protected override void Draw(GameTime gameTime)
    {
        GraphicsDevice.Clear(Color.CornflowerBlue);
        _spriteBatch.Begin();
        // _spriteBatch.Draw(...);
        _spriteBatch.End();
        base.Draw(gameTime);
    }
}
```

## Architecture / How It Works

The public API is the XNA 4.0 surface. Beneath it, each platform target is a separate build of `MonoGame.Framework` that binds the same API to a concrete graphics backend: OpenGL (`DesktopGL`) or DirectX (`WindowsDX`) on desktop, OpenGL ES on mobile, and native SDKs on consoles. You pick the backend by choosing the project template/NuGet package, not by a runtime flag, so a DesktopGL and a WindowsDX build of the same game are different binaries.

The other half of the framework is the **content pipeline**. MonoGame does not load PNGs, FBX, or WAV files at runtime. Instead, `mgcb` (the MonoGame Content Builder) processes source assets at build time into `.xnb` binary blobs, and `ContentManager.Load<T>()` reads those blobs. The pipeline is configured through a `.mgcb` file, edited either by hand or in the `mgcb-editor` GUI. Textures get premultiplied/compressed, shaders (`.fx`) are compiled by `mgfxc`, models are packed. This is powerful for shipping but is the single most common source of new-user confusion, because "the file is in my folder but won't load" almost always means it was never added to the content project.

Shaders are authored in HLSL and compiled by `mgfxc` to a MonoGame-specific bytecode (`MGFX`). On non-DirectX backends this goes through a cross-compilation step, historically via the MojoShader lineage, so HLSL features do not all map cleanly to GLSL targets. There is no runtime shader compilation from source strings the way some engines allow.

Input, audio (XACT and the simpler `SoundEffect`/`Song` APIs), math types (`Vector2/3`, `Matrix`, `Rectangle`), and the game timer are all carried forward from XNA with minimal change.

## Production Notes

- **The content pipeline is the footgun.** Assets must be added to a `.mgcb` project and built; a missing entry produces a runtime `ContentLoadException`, not a build error. Shipping raw files and loading them yourself (bypassing `.xnb`) is possible but off the happy path.
- **Backend choice is a commitment.** DesktopGL (OpenGL, one binary for Windows/Linux/macOS) versus WindowsDX (DirectX, Windows-only, better on some Windows GPUs) is decided up front. Shader effects and a few features behave differently across backends, so cross-backend testing is not optional.
- **No engine services.** Physics (nkast/Aether.Physics2D, or a Box2D binding), UI, ECS, tweening, and level editing are all third-party or hand-rolled. Budget for gluing an ecosystem together.
- **macOS/iOS is the fragile edge.** Apple targets have historically had the most build friction (Xamarin/.NET for Apple platforms churn, code-signing, OpenGL deprecation on macOS). The framework still runs on OpenGL on macOS, which Apple has deprecated; a Vulkan/Metal path is only in experimental/preview form as of the 3.8.5 line[^4].
- **Console access is gated.** PlayStation, Xbox (GDKX/XDK), and Switch builds require registered-developer status and are distributed privately, not from this repo[^5].
- **Slow-moving releases.** MonoGame ships infrequently; the 3.8.x line has spanned years. Fixes often land on `develop` long before a tagged NuGet release, so serious users sometimes build from source.

## When to Use / When Not

**Use when:**
- You want C#, full control of the render loop, and a minimal dependency surface.
- You are porting or continuing XNA code, or following XNA-era learning material.
- You are building a 2D game (or modest 3D) and prefer writing systems yourself over fighting an editor.
- Long-term maintainability and knowing exactly what your code does matter more than iteration speed.

**Avoid when:**
- You want an editor, visual scene composition, or drag-and-drop workflows — use a full engine.
- You need built-in physics, ECS, UI, lighting, or asset importing without assembling them.
- You are doing heavy 3D with modern rendering (PBR, deferred, compute-heavy) and don't want to write the renderer.
- Your team is non-programmer-heavy (designers, artists needing tooling).

## Alternatives

- FNA-XNA/FNA — a more conservative XNA re-implementation focused on faithful accuracy and shipping existing games; use instead when you want XNA parity over MonoGame's evolving extras.
- godotengine/godot — full engine with editor, scenes, physics, and scripting; use when you want batteries included and tooling, not a bare framework.
- libgdx/libgdx — comparable code-first, no-editor framework in the Java/Kotlin world; use when you prefer the JVM.
- flibitijibibo or nkast/XNAGameStudio ecosystem aside, Unity — use when you need a mature commercial engine, asset store, and non-coder workflows and accept a heavier runtime.
- love2d/love — minimal code-first 2D framework in Lua; use when you want the same "just a loop" philosophy with a lighter scripting language.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011-04 | Repo created as the successor to XNA Touch / early XNA ports[^6]. |
| 3.0 | 2013 | First unified cross-platform release after Microsoft ended XNA[^1]. |
| 3.5 | 2016 | Reworked content pipeline and tooling. |
| 3.6 | 2017 | Effect/shader and platform improvements. |
| 3.7 | 2018 | NuGet-first distribution maturing. |
| 3.8 | 2020 | Move to .NET Core / `dotnet` CLI templates, `mgcb-editor` as a dotnet tool. |
| 3.8.1–3.8.2 | 2022–2024 | .NET 6/8 targets, tooling and platform fixes. |
| 3.8.5 (preview) | 2026 | Experimental Vulkan / DirectX 12 backends in preview[^4]. |

## References

[^1]: Microsoft discontinued XNA Game Studio; MonoGame continued the API. Background: "MonoGame" — Wikipedia. https://en.wikipedia.org/wiki/MonoGame
[^2]: MonoGame showcase of shipped titles. https://monogame.net/showcase/
[^3]: MonoGame Foundation / project governance and donations. https://monogame.net/donate/
[^4]: README "Supported Platforms" note — Vulkan and DirectX 12 in preview for 3.8.5; experimental backends available to source users. https://github.com/MonoGame/MonoGame
[^5]: Console access for registered developers. https://docs.monogame.net/articles/console_access.html
[^6]: Repository created 2011-04-07 (GitHub API). https://github.com/MonoGame/MonoGame

## Tags

csharp, dotnet, game-framework, game-development, xna, cross-platform, 2d, 3d, graphics, opengl, content-pipeline
