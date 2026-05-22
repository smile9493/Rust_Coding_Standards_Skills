# 01 — Bevy ECS Architecture

Status: **Core** | Prerequisites: rust-architecture-guide §09-data-architecture

## Overview

Bevy ECS is the foundation of Rust game development. ECS separates identity (Entity), state (Component), and behavior (System). There are no classes, no inheritance trees, no virtual dispatch — just data and functions.

## Entity

Entity is a lightweight identifier. It contains no data, no behavior. An Entity is alive until `Commands::entity(id).despawn()`.

```rust
use bevy::prelude::*;

fn spawn_player(mut commands: Commands) {
    commands.spawn((
        Player,
        Name("Hero".into()),
        Transform::from_xyz(0.0, 0.0, 0.0),
        Velocity(Vec3::ZERO),
        Health { current: 100, max: 100 },
    ));
}
```

## Component

Component is a plain data struct that derives `Component`. It is stored in contiguous archetype storage (SoA layout) for cache-friendly iteration. Components must be `Send + Sync` if accessed across threads.

```rust
#[derive(Component)]
struct Player;

#[derive(Component)]
struct Name(&'static str);

#[derive(Component, Clone, Copy)]
struct Velocity(Vec3);

#[derive(Component, Clone, Copy)]
struct Health {
    current: f32,
    max: f32,
}

#[derive(Component)]
struct DamageFlash {
    timer: Timer,
    color: Color,
}

#[derive(Bundle)]
struct PlayerBundle {
    player: Player,
    name: Name,
    transform: Transform,
    velocity: Velocity,
    health: Health,
}
```

### Component Best Practices

- Keep components small. One concern per component. `Position` + `Velocity` is better than `MovementState { pos, vel, acc, drag }`.
- Use `#[derive(Component)]` macro. For marker/tag components, use a unit struct: `#[derive(Component)] struct Enemy;`.
- Use `#[require(Transform)]` when a component always needs another component present.

```rust
#[derive(Component)]
#[require(Transform)]
struct Projectile {
    direction: Vec3,
    speed: f32,
    lifetime: Timer,
}
```

## System

System is a function that iterates queries. Systems run in parallel when their queries do not conflict (Rust borrow rules enforced by the scheduler).

```rust
fn movement_system(
    time: Res<Time>,
    mut query: Query<(&mut Transform, &Velocity)>,
) {
    let dt = time.delta_secs();
    for (mut transform, velocity) in query.iter_mut() {
        transform.translation += velocity.0 * dt;
    }
}

fn health_system(
    mut commands: Commands,
    query: Query<(Entity, &Health)>,
) {
    for (entity, health) in query.iter() {
        if health.current <= 0.0 {
            commands.entity(entity).despawn_recursive();
        }
    }
}

fn damage_flash_system(
    time: Res<Time>,
    mut meshes: ResMut<Assets<Mesh>>,
    mut query: Query<(&mut DamageFlash, &Handle<Mesh>)>,
) {
    for (mut flash, mesh_handle) in query.iter_mut() {
        flash.timer.tick(time.delta());
        if flash.timer.just_finished() {
            let mesh = meshes.get_mut(mesh_handle).unwrap();
            mesh.remove_attribute(Mesh::ATTRIBUTE_COLOR);
        }
    }
}
```

## Resource

Resource is a globally unique piece of data. Not associated with any Entity. Accessed via `Res<T>` (read) or `ResMut<T>` (write). Resources are always inserted/removed in exclusive (sequential) systems.

```rust
#[derive(Resource)]
struct GameState {
    score: u32,
    wave: u32,
    difficulty_multiplier: f32,
}

#[derive(Resource, Default)]
struct SpawnTimer(Timer);

fn spawn_enemy_system(
    time: Res<Time>,
    mut game_state: ResMut<GameState>,
    mut spawn_timer: ResMut<SpawnTimer>,
    mut commands: Commands,
) {
    spawn_timer.0.tick(time.delta());
    if spawn_timer.0.just_finished() {
        game_state.wave += 1;
        let count = (game_state.wave as f32 * game_state.difficulty_multiplier) as u32;
        for i in 0..count {
            commands.spawn((Enemy, Transform::from_xyz(i as f32 * 2.0, 0.0, 0.0)));
        }
    }
}
```

## Query

Query is the primary data access pattern. Type-safe, compile-time verified.

```rust
fn complex_query_system(
    query: Query<(
        Entity,
        &Transform,
        &Health,
        Option<&Shield>,
        Without<Dead>,
    )>,
) {
    for (entity, transform, health, shield, _) in query.iter() {
        let effective_hp = health.current + shield.map(|s| s.value).unwrap_or(0.0);
        if effective_hp <= 0.0 {
            info!("Entity {:?} at {:?} died", entity, transform.translation);
        }
    }
}
```

### Query Filter Types

| Filter | Effect |
|--------|--------|
| `With<T>` | Entity must have component T |
| `Without<T>` | Entity must not have component T |
| `Changed<T>` | Component T was mutated this frame |
| `Added<T>` | Component T was added this frame |

## Commands

Commands are deferred mutations. They buffer entity operations and apply them at the end of the stage. This prevents borrowing conflicts during system execution.

```rust
fn spawn_bullet_system(
    mut commands: Commands,
    input: Res<ButtonInput<MouseButton>>,
    camera_query: Query<&Transform, With<Camera>>,
) {
    let camera_transform = camera_query.single();
    if input.just_pressed(MouseButton::Left) {
        commands.spawn((
            Bullet,
            Transform::from_translation(camera_transform.translation),
            Velocity(camera_transform.forward().as_vec3() * 50.0),
            LifeTimer(Timer::from_seconds(3.0, TimerMode::Once)),
        ));
    }
}
```

## App & Plugin

Plugin is the modular composition unit. Each game system is a plugin.

```rust
struct PlayerPlugin;

impl Plugin for PlayerPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(Startup, spawn_player)
            .add_systems(Update, (player_movement, player_shoot))
            .add_systems(PostUpdate, player_animation)
            .insert_resource(PlayerSettings::default())
            .add_event::<PlayerDiedEvent>();
    }
}

struct SettingsPlugin;

impl Plugin for SettingsPlugin {
    fn build(&self, app: &mut App) {
        app.add_plugins((PlayerPlugin, EnemyPlugin, CollisionPlugin));
    }
}
```

## Events

Events are one-to-many message passing. Writers push events; readers drain events. Events are cleared at the end of each frame.

```rust
#[derive(Event)]
struct PlayerDiedEvent {
    position: Vec3,
    killed_by: Option<Entity>,
}

#[derive(Event)]
struct ScoreChangedEvent {
    new_score: u32,
    delta: i32,
}

fn emit_death_event(
    mut death_writer: EventWriter<PlayerDiedEvent>,
    query: Query<(&Transform, &Health), With<Player>>,
) {
    for (transform, health) in query.iter() {
        if health.current <= 0.0 {
            death_writer.send(PlayerDiedEvent {
                position: transform.translation,
                killed_by: None,
            });
        }
    }
}

fn handle_death_event(
    mut death_reader: EventReader<PlayerDiedEvent>,
    mut commands: Commands,
) {
    for event in death_reader.read() {
        commands.spawn((
            Explosion,
            Transform::from_translation(event.position),
        ));
    }
}
```

## Change Detection

Bevy automatically tracks component mutations. Use `Changed<T>` to react only to modified data.

```rust
fn debug_health_changes(query: Query<&Health, Changed<Health>>) {
    for health in query.iter() {
        info!("Health changed: {}/{}", health.current, health.max);
    }
}
```

## System Ordering

Systems can be explicitly ordered when data dependency requires it:

```rust
app.add_systems(
    Update,
    (
        apply_velocity,
        detect_collisions,
        handle_collisions.after(detect_collisions),
        update_health.after(handle_collisions),
        despawn_dead.after(update_health),
    ).chain(),
);
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **No OOP hierarchies** | No class inheritance. Components are flat tags. Systems iterate queries. |
| **No direct World access in systems** | Use `Commands`, `Query`, `Res`, `EventWriter`. Raw `World` access is for bootstrap only. |
| **Components must be `Send + Sync`** | Systems run in parallel. Non-thread-safe components cause runtime panics. |
| **No heavy computation in `Query::single()` paths** | `single()` panics on mismatch. Validate entity existence with `get_single()` in production. |
| **Events are per-frame** | Events are drained each frame. Do not assume persistence across frames. |
| **Commands are deferred** | Entities spawned via `Commands` do not exist until the next stage. Do not query them in the same system. |

## References

- [Bevy ECS Book](https://bevyengine.org/learn/book/getting-started/ecs/)
- [Bevy System Ordering](https://bevyengine.org/learn/book/programming/systems/)
- rust-architecture-guide §09-data-architecture (Data-Oriented Design)