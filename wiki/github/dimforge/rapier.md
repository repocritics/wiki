# dimforge/rapier

> 2D and 3D rigid-body physics engines in pure Rust — the de facto physics layer for the Rust gamedev and WASM ecosystems.

[GitHub repo](https://github.com/dimforge/rapier) ·
[Official website](https://rapier.rs) ·
[License: Apache-2.0](https://github.com/dimforge/rapier/blob/master/LICENSE)

## Overview

Rapier is a family of rigid-body physics engines — `rapier2d`, `rapier3d`, and their `-f64` double-precision variants — written in Rust by Dimforge, the one-person-plus-contributors organization of Sébastien Crozet. It succeeded his earlier `nphysics`/`ncollide` stack in August 2020[^1] and sits on top of two sibling Dimforge crates: `nalgebra` (linear algebra) and `parry` (collision detection). Unlike C++ engines wrapped for Rust, Rapier is pure Rust end to end: no FFI, no build-system friction, first-class WASM.

That last property explains its reach beyond the star count (5.5k as of mid-2026). The official `rapier.js` WASM bindings[^2] made it one of the most common physics engines in the three.js/React Three Fiber world (`pmndrs/react-three-rapier`), and `bevy_rapier`[^3] made it the default answer for Bevy games for years. The defining tension: Rapier is a serious, benchmark-competitive engine maintained at bus-factor-one, still pre-1.0 after almost six years, with a history of both year-long lulls and monthly breaking releases. Since 2025 the project has been pivoting toward robotics — MJCF (MuJoCo format) loading via `mjcf-rs`/`rapier3d-mjcf` and official Python bindings — with AI-assisted coding explicitly used (and disclosed) for those newer crates[^4].

## Getting Started

```bash
cargo add rapier3d   # or rapier2d / rapier3d-f64 / rapier2d-f64
```

```rust
use rapier3d::prelude::*;

fn main() {
    let mut bodies = RigidBodySet::new();
    let mut colliders = ColliderSet::new();

    // Static ground.
    colliders.insert(ColliderBuilder::cuboid(100.0, 0.1, 100.0).build());

    // Bouncing ball, 10 m up.
    let ball = bodies.insert(
        RigidBodyBuilder::dynamic().translation(vector![0.0, 10.0, 0.0]).build(),
    );
    colliders.insert_with_parent(
        ColliderBuilder::ball(0.5).restitution(0.7).build(), ball, &mut bodies,
    );

    // Rapier owns no world object — you own every structure and pass them in.
    let gravity = vector![0.0, -9.81, 0.0];
    let params = IntegrationParameters::default();
    let mut pipeline = PhysicsPipeline::new();
    let mut islands = IslandManager::new();
    let mut broad_phase = DefaultBroadPhase::new();
    let mut narrow_phase = NarrowPhase::new();
    let mut impulse_joints = ImpulseJointSet::new();
    let mut multibody_joints = MultibodyJointSet::new();
    let mut ccd = CCDSolver::new();

    for _ in 0..200 {
        pipeline.step(&gravity, &params, &mut islands, &mut broad_phase,
            &mut narrow_phase, &mut bodies, &mut colliders,
            &mut impulse_joints, &mut multibody_joints, &mut ccd,
            &(), &());
        println!("ball y = {}", bodies[ball].translation().y);
    }
}
```

The exact `step()` arity shifts between 0.x releases (query pipeline, hooks, and event handler arguments have moved); check the user guide for your pinned version[^5].

## Architecture / How It Works

Rapier is data-oriented rather than object-oriented. There is no `World` god object: the user owns a set of arena-style containers (`RigidBodySet`, `ColliderSet`, joint sets, `IslandManager`, broad/narrow phase state) and hands them to `PhysicsPipeline::step()` each tick. Handles into these arenas (generational indices) replace pointers. This maps cleanly onto ECS engines and onto serde: the entire simulation state is plain data, so full snapshot/restore of a physics world is a supported, first-class operation — which is what makes rollback netcode and lockstep determinism practical.

Layers, bottom-up:

- **`nalgebra`** — vectors, isometries, SIMD via `simba`.
- **`parry`** (2D/3D) — shapes, SAP-based broad phase, narrow phase contact manifolds, ray/shape casting, scene queries. Usable standalone[^6].
- **`rapier`** — island management, impulse-based iterative constraint solver, impulse joints and reduced-coordinate multibody joints, CCD, sleeping, optional `rayon` parallelism, a built-in kinematic character controller.

The solver was rewritten for 0.18 (January 2024) around a small-steps / "TGS-soft"-style approach — more, cheaper substeps with soft constraints — which changed both stability characteristics and joint tuning semantics relative to the 0.17 era[^7]. The `enhanced-determinism` feature trades SIMD for bit-identical results across platforms that comply with IEEE 754-2008.

One dimension and one scalar type per crate: 2D vs 3D and f32 vs f64 are separate compiled crates, not runtime switches. Mixing 2D and 3D in one binary means compiling two engines.

## Production Notes

- **Pre-1.0 churn is real.** After a near-dormant 2023 (0.17 in January 2023, 0.18 in January 2024), the cadence flipped to roughly monthly minor releases through 2025–2026 (0.23 through 0.34 in eighteen months)[^8] — most with breaking API changes. Pin exact versions; budget upgrade time. Downstream wrappers (`bevy_rapier` especially) lag, so your Bevy version, wrapper version, and Rapier version form a compatibility triangle you don't fully control.
- **Bus factor.** Dimforge is essentially one maintainer with sponsorship funding. The 2023 lull demonstrated what that means in practice. The 2025 robotics pivot means core gamedev issue triage now competes with new surface area (MJCF, Python bindings).
- **Units footgun.** Rapier assumes SI units (meters, kilograms, seconds). Feeding pixel-scale coordinates into `rapier2d` — the classic mistake when porting from pixel-based 2D frameworks — produces jittery, exploding simulations. Keep bodies at meter-ish magnitudes (~0.1–10) and convert at render time.
- **Determinism costs.** `enhanced-determinism` disables SIMD and is incompatible with the `parallel` feature; expect a measurable throughput hit. Without it, results differ across platforms and sometimes across compiler upgrades.
- **Solver behavior shifts.** If you tuned joint stiffness/damping or stacking behavior on ≤0.17, the 0.18 solver rewrite changed the feel; retuning was widely needed. Treat solver-affecting upgrades as gameplay changes, not dependency bumps.
- **Scope limits.** Rigid bodies only: no cloth, no soft bodies; fluids live in the separate (and less maintained) `dimforge/salva`. Trimesh-vs-trimesh dynamic collision is a known weak spot common to impulse engines — prefer convex decomposition for dynamic bodies.
- **WASM.** `rapier.js` ships the engine as a WASM blob behind a generated JS API; it adds on the order of a megabyte to bundles, and its API tracks the Rust releases with the same churn.

## When to Use / When Not

**Use when:**
- You're building a game or simulation in Rust and want physics without FFI (Bevy via `bevy_rapier`, Fyrox, custom engines).
- You need physics in the browser: `rapier.js` is the strongest WASM-native option in the three.js ecosystem.
- You need snapshot/rollback or cross-platform determinism (lockstep networking) — Rapier's serializable state is a genuine differentiator.
- You want 2D and 3D from one API family, with an f64 option for large worlds.

**Avoid when:**
- You need soft bodies, cloth, destruction, or vehicle stacks out of the box — Jolt, PhysX, or an engine-integrated solution cover more ground.
- You need robotics-grade articulated dynamics and contact-rich manipulation today — MuJoCo remains the reference; Rapier's robotics story is young.
- You can't absorb breaking changes every few months — the pre-1.0 cadence is a standing cost.
- You're in Unity/Unreal/Godot: use the engine's built-in physics; wrapping Rapier buys you little there.

## Alternatives

- jrouwe/JoltPhysics — modern multicore C++ engine (Horizon Forbidden West); use instead when you need AAA-scale 3D features and C++ is acceptable.
- Jondolf/avian — ECS-native Rust physics for Bevy (formerly bevy_xpbd); use instead when you want physics living inside Bevy's ECS rather than wrapped alongside it.
- erincatto/box2d — the canonical 2D engine; use instead for 2D in C/C++ ecosystems or when you want two decades of tuning heritage.
- google-deepmind/mujoco — use instead for robotics simulation, RL training, and contact-rich articulated dynamics.
- bulletphysics/bullet3 — aging but ubiquitous C++ engine with pybullet; use instead when Python robotics tooling and ecosystem maturity matter more than speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2020-08-19 | Initial release; Dimforge's successor to nphysics[^1]. |
| 0.17 | 2023-01-15 | Last release before a year-long maintenance lull[^8]. |
| 0.18 | 2024-01-24 | Solver rewrite: small-steps, soft-constraint style[^7]. |
| 0.23 | 2025-01-08 | Start of the fast (roughly monthly) release cadence. |
| 0.27–0.31 | 2025-07 – 2025-11 | Sustained monthly breaking releases[^8]. |
| 0.34 | 2026-07-04 | Current line; era of MJCF + Python robotics bindings[^4]. |

## References

[^1]: rapier3d 0.1.0 on crates.io — 2020-08-19. https://crates.io/crates/rapier3d/0.1.0
[^2]: dimforge/rapier.js — official JavaScript/WASM bindings. https://github.com/dimforge/rapier.js
[^3]: dimforge/bevy_rapier — official Bevy plugin. https://github.com/dimforge/bevy_rapier
[^4]: Rapier README — Python bindings and AI coding policy sections. https://github.com/dimforge/rapier#readme
[^5]: Rapier user guide. https://rapier.rs/docs/
[^6]: dimforge/parry — collision detection library underlying Rapier. https://github.com/dimforge/parry
[^7]: Rapier CHANGELOG, v0.18.0 — 2024-01-24. https://github.com/dimforge/rapier/blob/master/CHANGELOG.md
[^8]: rapier3d version history on crates.io. https://crates.io/crates/rapier3d/versions

## Tags

rust, physics-engine, game-development, 2d, 3d, collision-detection, simulation, robotics, wasm, determinism, bevy
