# 05 — Shader Programming with WGSL & naga

Status: **Core** | Prerequisites: 03-gpu-wgpu.md, 04-rendering-pipeline.md

## Overview

Shaders are GPU programs written in WGSL (WebGPU Shading Language). naga is the Rust shader compiler that translates GLSL/SPIR-V/MSL into WGSL. Bevy uses WGSL natively, with naga for cross-compilation.

## WGSL Entry Points

Three entry point types: `@vertex`, `@fragment`, `@compute`. Each has a declared stage annotation.

### Vertex Shader

```wgsl
struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) normal: vec3<f32>,
    @location(2) uv: vec2<f32>,
    @location(3) tangent: vec4<f32>,
}

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) world_position: vec3<f32>,
    @location(1) world_normal: vec3<f32>,
    @location(2) uv: vec2<f32>,
}

struct Camera {
    view_proj: mat4x4<f32>,
    view_position: vec3<f32>,
}

@group(0) @binding(0) var<uniform> camera: Camera;

@vertex
fn vertex_main(in: VertexInput) -> VertexOutput {
    var out: VertexOutput;
    out.clip_position = camera.view_proj * vec4(in.position, 1.0);
    out.world_position = in.position;
    out.world_normal = in.normal;
    out.uv = in.uv;
    return out;
}
```

### Fragment Shader

```wgsl
struct Material {
    base_color: vec4<f32>,
    metallic: f32,
    roughness: f32,
    emissive: vec3<f32>,
}

@group(0) @binding(1) var base_color_texture: texture_2d<f32>;
@group(0) @binding(2) var base_color_sampler: sampler;
@group(1) @binding(0) var<uniform> material: Material;

@fragment
fn fragment_main(in: VertexOutput) -> @location(0) vec4<f32> {
    let sampled_color = textureSample(base_color_texture, base_color_sampler, in.uv);
    let final_color = sampled_color * material.base_color;

    let N = normalize(in.world_normal);
    let L = normalize(vec3<f32>(1.0, 1.0, 0.0));

    let NdotL = max(dot(N, L), 0.0);
    let ambient = vec3<f32>(0.05);
    let diffuse = final_color.rgb * NdotL;
    let lit = ambient + diffuse;

    return vec4<f32>(lit, final_color.a);
}
```

### Compute Shader

```wgsl
struct Particle {
    position: vec3<f32>,
    velocity: vec3<f32>,
    lifetime: f32,
    age: f32,
}

@group(0) @binding(0) var<storage, read_write> particles: array<Particle>;

struct SimulationParams {
    delta_time: f32,
    gravity: vec3<f32>,
    emitter_position: vec3<f32>,
    spawn_rate: u32,
    max_particles: u32,
}

@group(0) @binding(1) var<uniform> params: SimulationParams;

@compute @workgroup_size(64)
fn compute_main(@builtin(global_invocation_id) global_id: vec3<u32>) {
    let index = global_id.x;

    if index >= params.max_particles {
        return;
    }

    var p = particles[index];

    if p.age >= p.lifetime {
        p.position = params.emitter_position;
        p.velocity = vec3<f32>(
            cos(f32(index) * 0.1) * 5.0,
            sin(f32(index) * 0.13) * 5.0 + 10.0,
            sin(f32(index) * 0.07) * 5.0,
        );
        p.lifetime = 2.0 + sin(f32(index) * 0.3) * 1.0;
        p.age = 0.0;
    }

    p.velocity += params.gravity * params.delta_time;
    p.position += p.velocity * params.delta_time;
    p.age += params.delta_time;

    particles[index] = p;
}
```

## Bevy Shader Integration

```rust
use bevy::{
    prelude::*,
    render::render_resource::{
        Shader, ShaderType, AsBindGroup,
    },
    reflect::TypePath,
};

#[derive(Asset, TypePath, AsBindGroup, Debug, Clone)]
struct ParticleMaterial {
    #[uniform(0)]
    base_color: LinearRgba,
    #[texture(1)]
    #[sampler(2)]
    particle_texture: Handle<Image>,
}

impl Material for ParticleMaterial {
    fn vertex_shader() -> ShaderRef {
        "shaders/particle.wgsl".into()
    }
    fn fragment_shader() -> ShaderRef {
        "shaders/particle.wgsl".into()
    }
}

fn setup_particle_material(
    mut materials: ResMut<Assets<ParticleMaterial>>,
    asset_server: Res<AssetServer>,
) {
    materials.add(ParticleMaterial {
        base_color: LinearRgba::WHITE,
        particle_texture: asset_server.load("textures/particle.png"),
    });
}
```

## naga Cross-Compilation

naga compiles shader source formats (GLSL, SPIR-V, MSL, WGSL) into WGSL or any backend format.

```rust
use naga::{
    front::glsl::{Frontend, Options as GlslOptions},
    back::wgsl,
    valid::{Validator, ValidationFlags, Capabilities},
};

fn compile_glsl_to_wgsl(glsl_source: &str, stage: naga::ShaderStage) -> Result<String, String> {
    let mut frontend = Frontend::default();
    let options = GlslOptions {
        stage,
        defines: Default::default(),
    };

    let module = frontend.parse(&options, glsl_source)
        .map_err(|e| format!("GLSL parse error: {e}"))?;

    let mut validator = Validator::new(ValidationFlags::all(), Capabilities::all());
    validator.validate(&module)
        .map_err(|e| format!("Validation error: {e}"))?;

    let wgsl_output = wgsl::write_string(&module, &Default::default())
        .map_err(|e| format!("WGSL write error: {e}"))?;

    Ok(wgsl_output)
}
```

## Shader Hot-Reload

Bevy's `AssetServer` watches for file changes when `watch_for_changes()` is enabled. Shader assets are automatically reloaded.

```rust
fn setup_shader_hot_reload(
    asset_server: Res<AssetServer>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    #[cfg(debug_assertions)]
    asset_server.watch_for_changes().unwrap();
}
```

## Compute Shader Dispatch (Particle System)

```rust
use bevy::render::render_resource::{
    BindGroup, BindGroupLayout, ComputePipeline, PipelineCache,
    binding_types::{storage_buffer_read_only, uniform_buffer},
};

#[derive(Resource)]
struct ComputeParticlesPipeline {
    bind_group_layout: BindGroupLayout,
    pipeline: CachedComputePipelineId,
}

fn queue_compute_particles(
    pipeline_cache: Res<PipelineCache>,
    pipeline: Res<ComputeParticlesPipeline>,
    particle_buffers: Res<ParticleBuffers>,
    mut commands: bevy::render::render_graph::RenderCommandBuffer,
) {
    let Some(compute_pipeline) = pipeline_cache.get_compute_pipeline(pipeline.pipeline) else {
        return;
    };

    commands.set_compute_pipeline(compute_pipeline);
    commands.set_bind_group(0, &particle_buffers.bind_group, &[]);
    commands.dispatch_workgroups(
        (particle_buffers.count as u32 + 63) / 64,
        1,
        1,
    );
}
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Shader hot-reload in dev builds** | Iteration speed. Without hot-reload, shader tweaks require full restart. |
| **Use `@group`/`@binding` explicitly** | Implicit bindings are fragile. Explicit annotations prevent binding mismatches. |
| **Compute workgroup size = multiple of 64** | GPU warp/wavefront size. Non-aligned sizes waste threads. |
| **No dynamic branching in fragment shaders** | GPUs execute both branches on all threads. Use `discard` or early-Z. |
| **Validate shaders via naga at build time** | Catch shader errors before runtime. CI pipeline should run naga validation. |
| **Prefer WGSL over GLSL** | WGSL is the WebGPU standard. naga translation introduces edge cases. |

## References

- [WGSL Specification](https://www.w3.org/TR/WGSL/)
- [naga (GitHub)](https://github.com/gfx-rs/naga)
- [Bevy Shader Examples](https://github.com/bevyengine/bevy/tree/main/examples/shader)