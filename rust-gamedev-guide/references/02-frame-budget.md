# 02 — Frame Budget & Fixed Timestep

Status: **Core** | Prerequisites: 01-ecs-bevy.md

## Overview

Frame budget is the maximum time a single frame can take. At 60 FPS, the budget is ~16.67ms. At 120 FPS, ~8.33ms. Every system in the update loop must fit within this budget. The frame budget is law — exceeding it means dropped frames, input lag, and stuttering.

## Fixed Timestep

Physics and gameplay logic must use a fixed timestep. This ensures deterministic simulation regardless of frame rate. Bevy's `FixedUpdate` schedule runs at a configurable fixed rate (default 64 Hz).

```rust
use bevy::prelude::*;

#[derive(Resource)]
struct PhysicsConfig {
    timestep: f64,
    max_sub_steps: u32,
}

fn configure_fixed_timestep(app: &mut App) {
    app.insert_resource(Time::<Fixed>::from_hz(50.0));
    app.add_systems(FixedUpdate, (
        physics_step,
        apply_velocity,
        resolve_collisions,
    ).chain());
}

fn physics_step(
    time: Res<Time>,
    fixed_time: Res<Time<Fixed>>,
    mut query: Query<(&mut Transform, &Velocity)>,
) {
    let fixed_dt = fixed_time.delta_secs();
    for (mut transform, velocity) in query.iter_mut() {
        transform.translation += velocity.0 * fixed_dt;
    }
}
```

## Interpolation Rendering

Physics runs at fixed rate; rendering runs at variable rate. Interpolate between two physics snapshots for smooth visuals.

```rust
#[derive(Component)]
struct InterpolatedTransform {
    previous: Vec3,
    current: Vec3,
}

#[derive(Component)]
struct Velocity(Vec3);

fn interpolate_transforms(
    fixed_time: Res<Time<Fixed>>,
    mut query: Query<(&mut Transform, &InterpolatedTransform)>,
) {
    let alpha = fixed_time.overstep_fraction() as f32;
    for (mut transform, interpolated) in query.iter_mut() {
        transform.translation = interpolated.previous.lerp(interpolated.current, alpha);
    }
}
```

## Frame Pacing

`Time::delta()` provides the time since the last frame. Use this for frame-rate independent smoothing — but never use it for physics.

```rust
fn camera_follow_system(
    time: Res<Time>,
    mut camera: Query<&mut Transform, With<MainCamera>>,
    target: Query<&Transform, (With<Player>, Without<MainCamera>)>,
) {
    let dt = time.delta_secs();
    let Ok(mut cam) = camera.get_single_mut() else { return };
    let Ok(target) = target.get_single() else { return };

    let target_pos = target.translation + Vec3::new(0.0, 10.0, -15.0);
    cam.translation = cam.translation.lerp(target_pos, dt * 5.0);
}
```

## Frame Budget Breakdown

A typical 16.67ms budget allocation:

| System | Budget | Notes |
|--------|--------|-------|
| Input processing | < 0.5ms | Buffer via `leafwing-input-manager` |
| Physics (FixedUpdate) | < 3ms | rapier broadphase + narrowphase |
| AI / Game Logic | < 3ms | Pathfinding, behavior trees, scoring |
| Scene Traversal | < 1ms | Visibility determination, frustum culling |
| GPU Command Building | < 2ms | Draw call encoding, buffer updates |
| Render Pass Overhead | < 1ms | Begin/end render passes |
| GPU Execution | async | GPU runs in parallel with next frame's CPU |
| **Total CPU Budget** | **< 12ms** | Leave 4ms margin for OS/variance |

## Profiling

```rust
use bevy::diagnostic::{FrameTimeDiagnosticsPlugin, LogDiagnosticsPlugin};

fn setup_profiling(app: &mut App) {
    app.add_plugins((
        FrameTimeDiagnosticsPlugin,
        LogDiagnosticsPlugin::default(),
    ));
}
```

## bevy_framepace Integration

`bevy_framepace` provides VSync control and frame rate limiting, critical for consistent frame times on variable-refresh-rate displays.

```rust
use bevy_framepace::{FramepacePlugin, FramepaceSettings, Limiter};

fn configure_framepace(mut settings: ResMut<FramepaceSettings>) {
    let mut limiter = Limiter::from_framerate(60.0);
    settings.limiter = limiter;
}
```

## Budget Enforcement

When a system exceeds its budget, apply mitigation strategies in priority order:

1. **Frustum cull** — Skip invisible objects (biggest win)
2. **LOD** — Lower detail at distance
3. **Reduce iteration count** — Cap entities processed per frame
4. **Async offload** — Move work to background thread via `AsyncComputeTaskPool`

```rust
use bevy::tasks::AsyncComputeTaskPool;

fn expensive_computation(
    query: Query<(Entity, &Transform), With<NeedsProcessing>>,
) {
    let pool = AsyncComputeTaskPool::get();
    for (entity, transform) in query.iter() {
        let pos = transform.translation;
        pool.spawn(async move {
            heavy_pathfinding(pos)
        }).detach();
    }
}
```

## CPU-GPU Overlap

The CPU builds commands for frame N+1 while the GPU renders frame N. This pipelining is essential for hitting the budget.

```
Frame N:   |--CPU work--|  |--GPU work--|
Frame N+1:                |--CPU work--|  |--GPU work--|
Frame N+2:                               |--CPU work--|  |--GPU work--|
```

```rust
fn staggered_upload(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
) {
    commands.spawn(SceneRoot((
        Transform::default(),
        Visibility::default(),
    )));
}
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Physics in `FixedUpdate` only** | Variable timestep = non-deterministic simulation |
| **Never `sleep()` or `spin_loop()` in systems** | Blocks entire frame. Use async or yield. |
| **Render interpolation uses `overstep_fraction()`** | Only interpolate in `Update`, not `FixedUpdate` |
| **Profile before optimizing** | `FrameTimeDiagnosticsPlugin` identifies the bottleneck |
| **Leave 4ms CPU margin** | OS scheduling variance. If full budget is 16ms, target 12ms CPU. |
| **No per-frame allocations** | Use `Vec::with_capacity` pre-sized. Reuse buffers across frames. |

## References

- [Bevy Time and Timers](https://bevyengine.org/learn/book/features/time/)
- [bevy_framepace](https://github.com/aevyrie/bevy_framepace)
- [Fix Your Timestep (Gaffer on Games)](https://gafferongames.com/post/fix_your_timestep/)