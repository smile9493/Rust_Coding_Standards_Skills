# 04 — Rendering Pipeline Architecture

Status: **Core** | Prerequisites: 03-gpu-wgpu.md

## Overview

The rendering pipeline transforms 3D/2D scene data into screen pixels. Key concerns: minimize draw calls (< 1000 per frame), eliminate invisible objects early (frustum culling), and reduce detail at distance (LOD).

## Render Graph

Bevy's render graph organizes rendering into passes. Each pass has input/output attachments. Passes run in dependency order.

```rust
use bevy::render::render_graph::{RenderGraph, RenderLabel};

#[derive(Debug, Hash, PartialEq, Eq, Clone, RenderLabel)]
enum MyRenderLabel {
    ShadowPass,
    GBufferPass,
    LightingPass,
    TonemappingPass,
}

fn setup_render_graph(
    render_app: &mut bevy::render::app::RenderApp,
) {
    render_app.add_render_graph_node::<ShadowPassNode>(MyRenderLabel::ShadowPass);
    render_app.add_render_graph_node::<GBufferNode>(MyRenderLabel::GBufferPass);
    render_app.add_render_graph_node::<LightingNode>(MyRenderLabel::LightingPass);
    render_app.add_render_graph_node::<TonemappingNode>(MyRenderLabel::TonemappingPass);

    let mut graph = render_app.world_mut().resource_mut::<RenderGraph>();
    graph.add_node_edge(MyRenderLabel::GBufferPass, MyRenderLabel::ShadowPass);
    graph.add_node_edge(MyRenderLabel::LightingPass, MyRenderLabel::GBufferPass);
    graph.add_node_edge(MyRenderLabel::TonemappingPass, MyRenderLabel::LightingPass);
}
```

## Frustum Culling

Eliminate objects outside the camera frustum before they reach the GPU. Bevy provides built-in frustum culling via `NoFrustumCulling` to opt out.

```rust
use bevy::render::primitives::Aabb;

fn manual_frustum_cull(
    camera_query: Query<(&GlobalTransform, &Frustum), With<Camera>>,
    object_query: Query<(Entity, &GlobalTransform, &Aabb), Without<NoFrustumCulling>>,
    mut visibility: Query<&mut Visibility>,
) {
    let Ok((cam_transform, frustum)) = camera_query.get_single() else { return };

    for (entity, obj_transform, aabb) in object_query.iter() {
        let world_min = obj_transform.transform_point(aabb.min());
        let world_max = obj_transform.transform_point(aabb.max());
        let world_aabb = Aabb::from_min_max(world_min, world_max);

        let visible = frustum.intersects_obb(
            &world_aabb,
            &obj_transform.affine(),
            true,
            false,
        );

        if let Ok(mut vis) = visibility.get_mut(entity) {
            *vis = if visible { Visibility::Inherited } else { Visibility::Hidden };
        }
    }
}
```

### Sphere Culling (Faster Than AABB)

For objects that are roughly spherical, sphere-frustum tests are cheaper:

```rust
#[derive(Component)]
struct BoundingSphere {
    center: Vec3,
    radius: f32,
}

fn sphere_frustum_cull(
    camera_query: Query<&Frustum, With<Camera>>,
    query: Query<(Entity, &GlobalTransform, &BoundingSphere), Without<NoFrustumCulling>>,
    mut visibility: Query<&mut Visibility>,
) {
    let Ok(frustum) = camera_query.get_single() else { return };

    for (entity, transform, sphere) in query.iter() {
        let world_center = transform.transform_point(sphere.center);
        let world_radius = sphere.radius * transform.scale().max_element();

        let visible = frustum.intersects_sphere(&world_center, world_radius, false);

        if let Ok(mut vis) = visibility.get_mut(entity) {
            *vis = if visible { Visibility::Inherited } else { Visibility::Hidden };
        }
    }
}
```

## LOD System

Level of Detail reduces mesh complexity based on distance from camera.

```rust
#[derive(Component)]
struct LodGroup {
    levels: Vec<LodLevel>,
}

struct LodLevel {
    distance: f32,
    mesh: Handle<Mesh>,
}

fn lod_selection_system(
    camera: Query<&GlobalTransform, With<Camera>>,
    mut query: Query<(&GlobalTransform, &LodGroup, &mut Handle<Mesh>)>,
    meshes: Res<Assets<Mesh>>,
) {
    let Ok(camera_transform) = camera.get_single() else { return };
    let camera_pos = camera_transform.translation();

    for (transform, lod_group, mut current_mesh) in query.iter_mut() {
        let distance = transform.translation().distance(camera_pos);

        let chosen = lod_group.levels.iter()
            .rev()
            .find(|level| distance >= level.distance)
            .unwrap_or(&lod_group.levels[0]);

        if meshes.get(&chosen.mesh).is_some() {
            *current_mesh = chosen.mesh.clone();
        }
    }
}
```

## Instancing

Draw many copies of the same mesh with per-instance data. Dramatically reduces draw call count.

```rust
#[derive(Component)]
struct InstanceData {
    transforms: Vec<InstanceTransform>,
    colors: Vec<[f32; 4]>,
}

#[derive(bytemuck::Pod, bytemuck::Zeroable, Clone, Copy)]
#[repr(C)]
struct InstanceTransform {
    model: [[f32; 4]; 3],
}

fn build_instance_buffer(
    instance_data: &InstanceData,
    device: &wgpu::Device,
) -> wgpu::Buffer {
    use wgpu::util::DeviceExt;

    device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
        label: Some("instance_buffer"),
        contents: bytemuck::cast_slice(&instance_data.transforms),
        usage: wgpu::BufferUsages::VERTEX | wgpu::BufferUsages::COPY_DST,
    })
}

fn draw_instanced(
    render_pass: &mut wgpu::RenderPass,
    vertex_count: u32,
    instance_count: u32,
) {
    render_pass.draw(0..vertex_count, 0..instance_count);
}
```

### Bevy Instancing via MaterialMesh2d

```rust
use bevy::sprite::{MaterialMesh2dBundle, Mesh2dHandle};

fn spawn_instanced_sprites(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    let mesh = meshes.add(Rectangle::new(16.0, 16.0));
    let material = materials.add(Color::WHITE);

    let instances_per_row = 100;
    let spacing = 20.0;

    for row in 0..100 {
        for col in 0..instances_per_row {
            commands.spawn(MaterialMesh2dBundle {
                mesh: Mesh2dHandle(mesh.clone()),
                material: material.clone(),
                transform: Transform::from_xyz(
                    col as f32 * spacing,
                    row as f32 * spacing,
                    0.0,
                ),
                ..default()
            });
        }
    }
}
```

## Draw Call Budget

Target: **< 1000 draw calls per frame** at 60 FPS. Each draw call has CPU overhead (state validation, command encoding).

### Draw Call Reduction Strategies

| Strategy | Impact |
|----------|--------|
| Instancing | 1 draw call for N identical meshes |
| Material sorting | Group same-material objects to reduce state changes |
| Atlas textures | One texture for many sprites — one material, one draw call |
| Mesh merging | Combine static geometry offline |
| Frustum culling | Skip draw calls for invisible objects |
| Occlusion culling | Skip objects hidden behind other objects |

```rust
fn count_draw_calls_system(
    mut diagnostics: ResMut<bevy::diagnostic::DiagnosticsStore>,
) {
    diagnostics.add_measurement(
        bevy::diagnostic::DiagnosticsStore::get_measurement_name("draw_calls"),
        || { /* hook into renderer stats */ },
    );
}
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Draw calls < 1000 per frame** | Beyond this, CPU command encoding becomes the bottleneck at 60 FPS. |
| **Frustum cull before dispatch** | Never send invisible geometry to the GPU. |
| **Sort by material/depth** | Minimize pipeline state changes; maximize early-Z rejection. |
| **LOD: reduce triangles at distance** | A mesh at 100m does not need 50K triangles. |
| **Batch static geometry** | Merge non-moving meshes into single draw calls. |
| **No per-object render passes** | One render pass per technique (opaque, transparent, post-process). |

## References

- [Bevy Render Architecture](https://bevyengine.org/learn/book/rendering/)
- [GPU-Driven Rendering Pipelines (SIGGRAPH)](https://advances.realtimerendering.com/s2015/aaltonenhaar_siggraph2015_combined_final_footer_220dpi.pdf)
- [Frustum Culling (Lighthouse3D)](https://www.lighthouse3d.com/tutorials/view-frustum-culling/)