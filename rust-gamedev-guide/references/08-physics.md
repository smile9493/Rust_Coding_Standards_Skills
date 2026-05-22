# 08 — Physics & Collision Detection

Status: **Expertise** | Prerequisites: 01-ecs-bevy.md, 02-frame-budget.md

## Overview

Physics simulation runs at a fixed timestep using `rapier`, a pure-Rust physics engine. Collision detection uses spatial partitioning (broadphase) + precise intersection tests (narrowphase). Red line: physics must run in `FixedUpdate` — variable timestep produces non-deterministic results.

## rapier Rigid Bodies

Rigid bodies represent physical objects. They can be dynamic (affected by forces), kinematic (user-controlled, no force response), or static (immovable).

```rust
use bevy::prelude::*;
use bevy_rapier3d::prelude::*;

fn spawn_physics_objects(mut commands: Commands) {
    commands.spawn((
        RigidBody::Dynamic,
        Collider::cuboid(0.5, 0.5, 0.5),
        Transform::from_xyz(0.0, 5.0, 0.0),
        Velocity {
            linvel: Vec3::new(2.0, 0.0, 0.0),
            angvel: Vec3::ZERO,
        },
        GravityScale(1.0),
        Damping {
            linear_damping: 0.1,
            angular_damping: 0.1,
        },
    ));

    commands.spawn((
        RigidBody::Fixed,
        Collider::cuboid(50.0, 0.1, 50.0),
        Transform::from_xyz(0.0, -0.1, 0.0),
    ));

    commands.spawn((
        RigidBody::KinematicPositionBased,
        Collider::capsule_y(1.0, 0.5),
        Transform::from_xyz(0.0, 1.0, 10.0),
        KinematicCharacterController::default(),
    ));
}
```

### Rigid Body Types

| Type | Moves | Responds to Forces | Typical Use |
|------|-------|--------------------|-------------|
| `Dynamic` | Yes | Yes | Physics objects, ragdolls |
| `KinematicVelocityBased` | Yes (by velocity) | No | Moving platforms, doors |
| `KinematicPositionBased` | Yes (by position) | No | Character controllers |
| `Fixed` | No | No | Ground, walls, static geometry |
| `Sensor` | No | Detects overlap only | Triggers, detection zones |

## Colliders

Colliders define the shape of a physical object. They can be attached to rigid bodies or exist standalone as sensors.

```rust
use bevy_rapier3d::geometry::Collider;

fn create_colliders(mut commands: Commands) {
    commands.spawn((
        RigidBody::Dynamic,
        Collider::ball(0.5),
        Transform::from_xyz(-2.0, 5.0, 0.0),
    ));

    commands.spawn((
        RigidBody::Dynamic,
        Collider::cuboid(0.3, 0.8, 0.3),
        Transform::from_xyz(0.0, 5.0, 0.0),
    ));

    commands.spawn((
        RigidBody::Dynamic,
        Collider::cylinder(0.5, 1.0),
        Transform::from_xyz(2.0, 5.0, 0.0),
    ));

    commands.spawn((
        RigidBody::Fixed,
        Collider::trimesh_from_bevy_mesh(
            &bevy::prelude::Mesh::from(Cuboid::new(100.0, 0.5, 100.0)),
        ).unwrap(),
    ));

    commands.spawn((
        RigidBody::Fixed,
        Collider::heightfield_from_bevy_mesh(
            &bevy::prelude::Mesh::from(TerrainConfig::default()),
        ).unwrap(),
    ));
}
```

## Joints

Joints connect two rigid bodies with movement constraints.

```rust
use bevy_rapier3d::dynamics::{
    ImpulseJoint, FixedJointBuilder, RevoluteJointBuilder,
    SphericalJointBuilder, PrismaticJointBuilder,
};

fn create_joints(mut commands: Commands) {
    let body_a = commands.spawn((
        RigidBody::Dynamic,
        Collider::cuboid(0.5, 0.2, 0.2),
        Transform::from_xyz(0.0, 2.0, 0.0),
    )).id();

    let body_b = commands.spawn((
        RigidBody::Dynamic,
        Collider::cuboid(0.2, 0.8, 0.2),
        Transform::from_xyz(0.0, 1.0, 0.0),
    )).id();

    let joint = RevoluteJointBuilder::new(Vec3::Y)
        .local_anchor1(Vec3::new(0.0, -0.2, 0.0))
        .local_anchor2(Vec3::new(0.0, 0.4, 0.0))
        .limits([-1.5, 1.5]);

    commands.spawn(ImpulseJoint::new(body_a, joint)).insert(body_b);
}
```

## CollisionGroups — Layered Filtering

`CollisionGroups` enables layer-based filtering. Objects on different layers do not collide.

```rust
use bevy_rapier3d::geometry::{CollisionGroups, Group};

const GROUP_PLAYER: Group = Group::GROUP_1;
const GROUP_ENEMY: Group = Group::GROUP_2;
const GROUP_PROJECTILE: Group = Group::GROUP_3;
const GROUP_WORLD: Group = Group::GROUP_4;

fn setup_collision_layers(mut commands: Commands) {
    commands.spawn((
        RigidBody::Dynamic,
        Collider::capsule_y(1.0, 0.3),
        CollisionGroups::new(
            GROUP_PLAYER,                     // membership
            GROUP_WORLD | GROUP_ENEMY | GROUP_PROJECTILE, // filter (what to collide with)
        ),
    ));

    commands.spawn((
        RigidBody::Dynamic,
        Collider::cuboid(0.5, 0.5, 0.5),
        CollisionGroups::new(
            GROUP_ENEMY,
            GROUP_WORLD | GROUP_PLAYER | GROUP_PROJECTILE,
        ),
    ));

    commands.spawn((
        RigidBody::KinematicVelocityBased,
        Collider::ball(0.1),
        CollisionGroups::new(
            GROUP_PROJECTILE,
            GROUP_WORLD | GROUP_PLAYER | GROUP_ENEMY,
        ),
    ));

    commands.spawn((
        RigidBody::Fixed,
        Collider::cuboid(50.0, 0.1, 50.0),
        CollisionGroups::new(
            GROUP_WORLD,
            Group::ALL,
        ),
    ));
}
```

## Collision Events

rapier emits collision events that can be queried each frame.

```rust
fn handle_collision_events(
    mut collision_events: EventReader<CollisionEvent>,
    mut health_query: Query<&mut Health>,
    damage_query: Query<&Damage>,
) {
    for event in collision_events.read() {
        match event {
            CollisionEvent::Started(e1, e2, _flags) => {
                if let (Ok(damage), Ok(mut health)) = (
                    damage_query.get(*e1),
                    health_query.get_mut(*e2),
                ) {
                    health.current -= damage.value;
                }
            }
            CollisionEvent::Stopped(e1, e2, _flags) => {
                info!("Collision ended between {:?} and {:?}", e1, e2);
            }
        }
    }
}
```

## Spatial Partitioning

rapier uses a two-phase collision detection pipeline:

### Broadphase — AABB Tree

Coarse spatial partitioning. Objects are indexed in a dynamic AABB tree. Broadphase eliminates pairs that cannot possibly intersect, reducing the number of narrowphase tests to O(n log n).

```rust
use bevy_rapier3d::pipeline::PhysicsHooks;

fn configure_physics(app: &mut App) {
    app.insert_resource(RapierConfiguration {
        timestep_mode: TimestepMode::Fixed {
            dt: 1.0 / 60.0,
            substeps: 1,
        },
        ..default()
    });
}
```

### Narrowphase — GJK/EPA

Precise intersection for convex shapes. GJK (Gilbert-Johnson-Keerthi) computes distance/separation. EPA (Expanding Polytope Algorithm) computes contact points and penetration depth for overlapping convex shapes.

```
Broadphase (AABB tree):
  All pairs → AABB overlap test → candidate pairs

Narrowphase (GJK/EPA):
  Candidate pair → convex intersection → contact manifold
```

## Fixed Timestep Enforcement

Physics must run at a fixed rate. rapier integrates with Bevy's `FixedUpdate` schedule. The `RapierPhysicsPlugin` defaults to 60 Hz.

```rust
use bevy_rapier3d::plugin::{RapierPhysicsPlugin, NoUserData};

fn setup_rapier(app: &mut App) {
    app.add_plugins(RapierPhysicsPlugin::<NoUserData>::default());
    app.insert_resource(RapierConfiguration {
        gravity: Vec3::new(0.0, -9.81, 0.0),
        timestep_mode: TimestepMode::Fixed {
            dt: 1.0 / 60.0,   // 60 Hz physics
            substeps: 1,
        },
        ..default()
    });
    app.add_systems(FixedUpdate, apply_external_forces);
}
```

## Ray Casting

```rust
fn ray_cast_system(
    rapier_context: Res<RapierContext>,
    camera_query: Query<&GlobalTransform, With<Camera>>,
) {
    let camera_transform = camera_query.single();
    let ray_pos = camera_transform.translation();
    let ray_dir = camera_transform.forward().as_vec3();

    let max_toi = 100.0;
    let solid = true;
    let filter = QueryFilter::default()
        .exclude_sensors()
        .groups(CollisionGroups::new(GROUP_PLAYER, GROUP_WORLD));

    if let Some((entity, toi)) = rapier_context.cast_ray(
        ray_pos, ray_dir, max_toi, solid, filter,
    ) {
        let hit_point = ray_pos + ray_dir * toi;
        info!("Hit {:?} at {:?}", entity, hit_point);
    }
}
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Physics in `FixedUpdate` only** | Variable timestep = non-deterministic simulation. Fixed = reproducible. |
| **Use `CollisionGroups` for layer filtering** | Avoids wasteful narrowphase tests. Layers are bitmasks, efficient. |
| **Prefer primitive colliders (cuboid/ball/capsule)** | Much faster than trimesh. Use trimesh only for static geometry. |
| **Keep substeps = 1 unless tunneling** | Each substep is a full physics step. More substeps = more CPU cost. |
| **Gravity at -9.81, not approximate values** | Consistency with real-world physics. Match art pipeline scale to this. |
| **No collider manipulation in `Update`** | Collider transforms are synced from rigid body positions in `FixedUpdate`. |

## References

- [rapier Documentation](https://rapier.rs/docs/)
- [bevy_rapier3d](https://docs.rs/bevy_rapier3d/latest/bevy_rapier3d/)
- [GJK Algorithm Explanation](https://dyn4j.org/2010/04/gjk-gilbert-johnson-keerthi/)
- [EPA Algorithm Explanation](https://dyn4j.org/2010/05/epa-expanding-polytope-algorithm/)