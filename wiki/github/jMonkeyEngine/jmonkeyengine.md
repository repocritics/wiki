# jMonkeyEngine/jmonkeyengine

> A scene-graph-based 3D game engine for the JVM — the most complete open-source path to shipping a 3D game without leaving Java.

[GitHub repo](https://github.com/jMonkeyEngine/jmonkeyengine) ·
[Official website](https://jmonkeyengine.org) ·
[License: BSD-3-Clause](https://github.com/jMonkeyEngine/jmonkeyengine/blob/master/LICENSE.md)

## Overview

jMonkeyEngine (jME) is a Java 3D game engine organized around a classic scene graph. The original engine dates to 2003 (Mark Powell); the current codebase, jME3, is a from-scratch rewrite begun around 2008–2009 and declared stable in 2014[^1]. Since the original founders moved on, it has been community-governed, with development coordinated through the hub forum[^2]. At ~4.3k stars it is a niche engine, but an actively maintained one — v3.8.0 is the current stable release and the repo sees commits weekly[^3].

The defining trait is that jME3 is a **library, not an editor-centric engine**. You write a Java (or Kotlin/Scala) application, add Gradle dependencies from Maven Central, and drive the engine in code. There is a NetBeans-derived SDK maintained in a separate repo, but it is optional and most current users skip it in favor of IntelliJ + Gradle. This makes jME appealing to programmers who dislike Unity/Godot's editor-first workflow — and unappealing to teams with artists and designers who need one.

The central tradeoff: you get JVM productivity, garbage collection, and a genuinely permissive BSD license against a renderer that is OpenGL-only (no Vulkan, no Metal, no D3D) and an asset/tooling ecosystem several orders of magnitude smaller than Unity's or Godot's. Several shipped commercial games exist (Mythruna, Spoxel, PirateHell)[^4], so the engine is proven for indie-scale 3D — but it is a "bring your own pipeline" proposition.

## Getting Started

Engine artifacts are on Maven Central; note the `-stable` suffix in version strings:

```groovy
dependencies {
    implementation "org.jmonkeyengine:jme3-core:3.8.0-stable"
    implementation "org.jmonkeyengine:jme3-desktop:3.8.0-stable"
    implementation "org.jmonkeyengine:jme3-lwjgl3:3.8.0-stable"
}
```

```java
import com.jme3.app.SimpleApplication;
import com.jme3.material.Material;
import com.jme3.math.ColorRGBA;
import com.jme3.scene.Geometry;
import com.jme3.scene.shape.Box;

public class Main extends SimpleApplication {
    public static void main(String[] args) {
        new Main().start();
    }

    @Override
    public void simpleInitApp() {
        Geometry geom = new Geometry("Box", new Box(1, 1, 1));
        Material mat = new Material(assetManager,
                "Common/MatDefs/Misc/Unshaded.j3md");
        mat.setColor("Color", ColorRGBA.Blue);
        geom.setMaterial(mat);
        rootNode.attachChild(geom);
    }
}
```

Do not build against the `master` branch — the README states plainly it is a development version not meant for production[^4]. Use a tagged stable release.

## Architecture / How It Works

- **Scene graph.** `Spatial` is the base type; `Node` (internal, has children) and `Geometry` (leaf, mesh + material) form the tree. Transforms cascade parent-to-child; culling is hierarchical bounding-volume based.
- **Materials.** A `Material` instantiates a `.j3md` material definition — a declarative file mapping named parameters to GLSL shader techniques. Stock definitions include `Unshaded`, `Lighting` (phong), and `PBRLighting`. Custom shaders mean writing a `.j3md` plus GLSL, not just dropping in shader files.
- **Behavior model.** Two attachment points: `AppState` (application-scoped logic, e.g. a physics state or game-mode state) and `Control` (per-spatial logic, e.g. an AI controller attached to a character). Both get `update(tpf)` callbacks from the single main loop. There is no ECS; this is a classic update-callback architecture.
- **Assets.** `AssetManager` loads models (glTF, OBJ, and the native `.j3o` binary format), textures, sounds, and fonts through a locator/loader registry. The recommended pipeline converts source models to `.j3o` at build time.
- **Modules.** The Gradle build splits the engine into `jme3-core`, `jme3-desktop`, `jme3-lwjgl`/`jme3-lwjgl3`, `jme3-android`, `jme3-ios`, `jme3-terrain`, `jme3-networking` (SpiderMonkey), `jme3-jbullet`/`jme3-bullet` (physics), `jme3-niftygui`, and `jme3-examples`. You depend only on what you use.
- **Backends.** Desktop rendering goes through LWJGL 2 or 3 (GLFW, OpenGL, OpenAL); Android uses OpenGL ES; iOS support exists but is the least exercised path[^4].
- **Threading.** The engine runs one render/update thread. The scene graph is not thread-safe: any modification from another thread must be posted via `application.enqueue(...)`. Violating this produces intermittent, hard-to-reproduce crashes — it is the canonical jME beginner bug.

## Production Notes

- **OpenGL-only renderer.** No Vulkan/Metal/D3D backends. On macOS you are capped at the deprecated OpenGL 4.1 core profile, which rules out compute shaders and some modern techniques there. This is the engine's largest strategic liability going forward.
- **Native buffer management.** Mesh and texture data live in direct NIO buffers outside the Java heap. Creating meshes dynamically without disposing them leaks native memory that the GC will not reliably reclaim; long-running procedural-geometry games need explicit buffer hygiene.
- **GC discipline still matters.** The JVM's GC handles most workloads fine, but per-frame allocation in `update()` loops (e.g. `new Vector3f()` in hot paths) causes pause hiccups. Idiomatic jME code reuses temp vectors; the engine provides `TempVars` for this.
- **Stock physics is dated.** `jme3-jbullet` (java port) and `jme3-bullet` lag upstream Bullet. The community standard is **stephengold/Minie**, a maintained drop-in replacement with native Bullet builds, soft bodies, and better debugging[^5]. New projects should start with Minie, not the bundled bindings.
- **GUI: skip Nifty.** The bundled `jme3-niftygui` integration is widely considered painful and effectively legacy. The community favorite is **Lemur** (jMonkeyEngine-Contributions/Lemur), a scene-graph-native GUI toolkit[^6].
- **SDK is optional and separate.** The NetBeans-based SDK lives in `jMonkeyEngine/sdk` on its own release cadence. Its main unique value is the `.j3o` scene composer and asset conversion; engine development itself works from any IDE.
- **Deployment.** Shipping means shipping a JVM: `jlink`/`jpackage` produce self-contained installers. Expect 40–80 MB baseline distribution size before assets.
- **Upgrade pains are moderate.** Point releases (3.5 → 3.6 → 3.7 → 3.8) have been conservative; the one historically disruptive migration was the 3.3 animation system replacement (`AnimComposer` replacing the legacy `Animation`/`AnimControl` classes), which required porting animated-model code.

## When to Use / When Not

**Use when:**
- Your team lives on the JVM and wants 3D without switching languages or toolchains.
- You prefer a code-first engine-as-library over an editor-centric workflow.
- You want a genuinely permissive license (BSD-3-Clause, no royalties, no seat fees) with full source.
- You are teaching 3D graphics/game architecture — the scene graph, material, and loop model are readable and well-documented[^2].

**Avoid when:**
- You need artists/designers working in an editor — Unity, Godot, and Unreal are categorically better tooled.
- You target consoles, or need Vulkan/Metal-class rendering features.
- You need a large asset marketplace or a hiring pool; jME's community is small (forum-centric, low thousands of active users).
- You are making a 2D game — jME can do it, but libGDX is leaner and better suited on the same JVM.

## Alternatives

- libgdx/libgdx — use instead for 2D or lower-level cross-platform Java (including HTML5 via GWT); framework rather than engine, no scene graph opinions.
- LWJGL/lwjgl3 — use instead when you want raw OpenGL/Vulkan bindings and will build your own engine layer.
- godotengine/godot — use instead when you need a full editor, 2D+3D, and a bigger community, and leaving Java is acceptable.
- bevyengine/bevy — use instead if you want a modern code-first engine and are willing to trade JVM for Rust and ECS architecture.
- Unity (closed source, not on this wiki) — use instead when asset-store leverage, console targets, and hiring pool outweigh licensing and openness.

## History

| Version | Date | Notes |
|---------|------|-------|
| jME 0.x–2.x | 2003–2008 | Original engine (Mark Powell); jME2 line now defunct[^1]. |
| 3.0 | 2014 | First stable jME3 — complete rewrite: shader-based materials, new scene graph[^1]. |
| 3.1 | 2017 | PBR materials and light probes; Gradle-based build[^3]. |
| 3.2 | 2018 | Incremental renderer and API improvements[^3]. |
| 3.3 | 2020 | New animation system (`AnimComposer`) replacing legacy animation classes[^3]. |
| 3.5 | 2022 | Renderer improvements; LWJGL 3 as the default desktop path matured[^3]. |
| 3.6 | 2023 | Maintenance release; dependency and platform updates[^3]. |
| 3.7 | 2024 | Maintenance release[^3]. |
| 3.8 | 2025 | Current stable line per project README[^4]. |

## References

[^1]: jMonkeyEngine website and project history. https://jmonkeyengine.org
[^2]: jMonkeyEngine wiki (docs and tutorials). https://jmonkeyengine.github.io/wiki
[^3]: GitHub releases (version dates). https://github.com/jMonkeyEngine/jmonkeyengine/releases
[^4]: Project README (games list, stack, master-branch warning). https://github.com/jMonkeyEngine/jmonkeyengine#readme
[^5]: stephengold/Minie — maintained Bullet physics for jME3. https://stephengold.github.io/Minie/minie/overview.html
[^6]: jMonkeyEngine-Contributions/Lemur — scene-graph-native GUI library. https://github.com/jMonkeyEngine-Contributions/Lemur

## Tags

java, game-engine, 3d, scene-graph, opengl, lwjgl, jvm, gamedev, cross-platform, android, desktop
