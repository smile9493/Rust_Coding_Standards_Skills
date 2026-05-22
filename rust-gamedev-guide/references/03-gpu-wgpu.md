# 03 — GPU Resource Lifecycle with wgpu

Status: **Core** | Prerequisites: 01-ecs-bevy.md

## Overview

GPU resources (buffers, textures, samplers) have strict lifetimes governed by the GPU queue. The CPU must not write to a buffer while the GPU is reading it. wgpu's ownership model encodes these rules at compile time through Rust borrows.

## Buffer

`wgpu::Buffer` is a contiguous region of GPU memory. Usage is declared at creation via `BufferUsages` bitflags.

```rust
use wgpu::{Buffer, BufferUsages, BufferDescriptor, BufferAddress};

fn create_vertex_buffer(device: &wgpu::Device, vertices: &[Vertex]) -> Buffer {
    device.create_buffer(&BufferDescriptor {
        label: Some("vertex_buffer"),
        size: (std::mem::size_of::<Vertex>() * vertices.len()) as BufferAddress,
        usage: BufferUsages::VERTEX | BufferUsages::COPY_DST,
        mapped_at_creation: true,
    })
}

fn create_uniform_buffer(device: &wgpu::Device) -> Buffer {
    device.create_buffer(&BufferDescriptor {
        label: Some("uniform_buffer"),
        size: std::mem::size_of::<Uniforms>() as BufferAddress,
        usage: BufferUsages::UNIFORM | BufferUsages::COPY_DST,
        mapped_at_creation: false,
    })
}

fn create_storage_buffer(device: &wgpu::Device, capacity: u64) -> Buffer {
    device.create_buffer(&BufferDescriptor {
        label: Some("instance_buffer"),
        size: capacity,
        usage: BufferUsages::STORAGE | BufferUsages::COPY_DST | BufferUsages::VERTEX,
        mapped_at_creation: false,
    })
}
```

### BufferUsages Flags

| Flag | Purpose |
|------|---------|
| `MAP_READ` | CPU reads from buffer |
| `MAP_WRITE` | CPU writes to buffer |
| `COPY_SRC` | Source for `copy_buffer_to_buffer`, `copy_buffer_to_texture` |
| `COPY_DST` | Destination for copy commands |
| `VERTEX` | Vertex buffer binding |
| `INDEX` | Index buffer binding |
| `UNIFORM` | Uniform buffer for bind groups |
| `STORAGE` | Storage (read/write) buffer for bind groups |
| `INDIRECT` | Indirect draw/dispatch arguments |

## Texture

`wgpu::Texture` represents GPU image data. `TextureView` provides a specific interpretation (format, dimension, mip level range).

```rust
use wgpu::{Texture, TextureView, TextureDescriptor, Extent3d, TextureDimension, TextureFormat, TextureUsages};

fn create_render_texture(
    device: &wgpu::Device,
    config: &wgpu::SurfaceConfiguration,
) -> (Texture, TextureView) {
    let size = Extent3d {
        width: config.width,
        height: config.height,
        depth_or_array_layers: 1,
    };

    let texture = device.create_texture(&TextureDescriptor {
        label: Some("render_target"),
        size,
        mip_level_count: 1,
        sample_count: 1,
        dimension: TextureDimension::D2,
        format: config.format,
        usage: TextureUsages::RENDER_ATTACHMENT | TextureUsages::TEXTURE_BINDING,
        view_formats: &[],
    });

    let view = texture.create_view(&wgpu::TextureViewDescriptor::default());
    (texture, view)
}

fn create_depth_texture(
    device: &wgpu::Device,
    config: &wgpu::SurfaceConfiguration,
) -> (Texture, TextureView) {
    let size = Extent3d {
        width: config.width,
        height: config.height,
        depth_or_array_layers: 1,
    };

    let texture = device.create_texture(&TextureDescriptor {
        label: Some("depth_texture"),
        size,
        mip_level_count: 1,
        sample_count: 1,
        dimension: TextureDimension::D2,
        format: TextureFormat::Depth32Float,
        usage: TextureUsages::RENDER_ATTACHMENT | TextureUsages::TEXTURE_BINDING,
        view_formats: &[],
    });

    let view = texture.create_view(&wgpu::TextureViewDescriptor::default());
    (texture, view)
}
```

## Sampler

Sampler controls how textures are sampled (filtering, addressing, anisotropy).

```rust
use wgpu::{Sampler, SamplerDescriptor, AddressMode, FilterMode, CompareFunction};

fn create_sampler(device: &wgpu::Device) -> Sampler {
    device.create_sampler(&SamplerDescriptor {
        label: Some("linear_sampler"),
        address_mode_u: AddressMode::ClampToEdge,
        address_mode_v: AddressMode::ClampToEdge,
        address_mode_w: AddressMode::ClampToEdge,
        mag_filter: FilterMode::Linear,
        min_filter: FilterMode::Linear,
        mipmap_filter: FilterMode::Linear,
        anisotropy_clamp: 16,
        compare: None,
        lod_min_clamp: 0.0,
        lod_max_clamp: f32::MAX,
    })
}

fn create_shadow_sampler(device: &wgpu::Device) -> Sampler {
    device.create_sampler(&SamplerDescriptor {
        label: Some("shadow_sampler"),
        address_mode_u: AddressMode::ClampToEdge,
        address_mode_v: AddressMode::ClampToEdge,
        address_mode_w: AddressMode::ClampToEdge,
        mag_filter: FilterMode::Linear,
        min_filter: FilterMode::Linear,
        mipmap_filter: FilterMode::Nearest,
        compare: Some(CompareFunction::LessEqual),
        ..Default::default()
    })
}
```

## Staging Belt

**Red Line**: Never map a GPU buffer directly on the render thread. The GPU may be using it. Use a staging belt — a ring buffer of CPU-writable staging buffers that are copied to GPU buffers via `copy_buffer_to_buffer`.

```rust
use wgpu::util::StagingBelt;

struct StagingManager {
    belt: StagingBelt,
    chunk_size: u64,
}

impl StagingManager {
    fn new(chunk_size: u64) -> Self {
        Self {
            belt: StagingBelt::new(chunk_size),
            chunk_size,
        }
    }

    fn upload_uniform<T: bytemuck::Pod>(
        &mut self,
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        encoder: &mut wgpu::CommandEncoder,
        destination: &Buffer,
        data: &T,
    ) {
        let size = std::mem::size_of::<T>() as u64;
        let mut view = self.belt.write_buffer(
            encoder,
            destination,
            0,
            wgpu::BufferSize::new(size).unwrap(),
            device,
        );
        view.copy_from_slice(bytemuck::bytes_of(data));
    }

    fn recall(&mut self) {
        self.belt.recall();
    }
}
```

## Bind Groups & Bind Group Layouts

Bind groups connect shader resources (buffers, textures, samplers) to the pipeline.

```rust
fn create_bind_group_layout(device: &wgpu::Device) -> wgpu::BindGroupLayout {
    device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
        label: Some("camera_bind_group_layout"),
        entries: &[
            wgpu::BindGroupLayoutEntry {
                binding: 0,
                visibility: wgpu::ShaderStages::VERTEX | wgpu::ShaderStages::FRAGMENT,
                ty: wgpu::BindingType::Buffer {
                    ty: wgpu::BufferBindingType::Uniform,
                    has_dynamic_offset: false,
                    min_binding_size: wgpu::BufferSize::new(
                        std::mem::size_of::<Uniforms>() as u64,
                    ),
                },
                count: None,
            },
            wgpu::BindGroupLayoutEntry {
                binding: 1,
                visibility: wgpu::ShaderStages::FRAGMENT,
                ty: wgpu::BindingType::Texture {
                    sample_type: wgpu::TextureSampleType::Float { filterable: true },
                    view_dimension: wgpu::TextureViewDimension::D2,
                    multisampled: false,
                },
                count: None,
            },
            wgpu::BindGroupLayoutEntry {
                binding: 2,
                visibility: wgpu::ShaderStages::FRAGMENT,
                ty: wgpu::BindingType::Sampler(wgpu::SamplerBindingType::Filtering),
                count: None,
            },
        ],
    })
}

fn create_bind_group(
    device: &wgpu::Device,
    layout: &wgpu::BindGroupLayout,
    uniform_buffer: &Buffer,
    texture_view: &TextureView,
    sampler: &Sampler,
) -> wgpu::BindGroup {
    device.create_bind_group(&wgpu::BindGroupDescriptor {
        label: Some("camera_bind_group"),
        layout,
        entries: &[
            wgpu::BindGroupEntry {
                binding: 0,
                resource: uniform_buffer.as_entire_binding(),
            },
            wgpu::BindGroupEntry {
                binding: 1,
                resource: wgpu::BindingResource::TextureView(texture_view),
            },
            wgpu::BindGroupEntry {
                binding: 2,
                resource: wgpu::BindingResource::Sampler(sampler),
            },
        ],
    })
}
```

## Queue Submission

All GPU commands are recorded into `CommandEncoder`, then submitted to the queue. Submissions are batched for efficiency.

```rust
fn render_frame(
    device: &wgpu::Device,
    queue: &wgpu::Queue,
    surface: &wgpu::Surface,
    config: &wgpu::SurfaceConfiguration,
    render_pass_fn: impl FnOnce(&mut wgpu::RenderPass),
) {
    let output = surface.get_current_texture()
        .expect("Failed to acquire swap chain texture");

    let view = output.texture.create_view(&wgpu::TextureViewDescriptor::default());

    let mut encoder = device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
        label: Some("render_encoder"),
    });

    {
        let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            label: Some("main_pass"),
            color_attachments: &[Some(wgpu::RenderPassColorAttachment {
                view: &view,
                resolve_target: None,
                ops: wgpu::Operations {
                    load: wgpu::LoadOp::Clear(wgpu::Color {
                        r: 0.1,
                        g: 0.2,
                        b: 0.3,
                        a: 1.0,
                    }),
                    store: wgpu::StoreOp::Store,
                },
            })],
            depth_stencil_attachment: None,
            timestamp_writes: None,
            occlusion_query_set: None,
        });

        render_pass_fn(&mut render_pass);
    }

    queue.submit(std::iter::once(encoder.finish()));
    output.present();
}
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **No direct GPU buffer mapping on render thread** | GPU may be reading. Use staging belt. |
| **Declare all BufferUsages at creation** | Buffer usage is immutable after creation. |
| **Bind group layout must match shader bindings exactly** | Mismatch causes validation errors. |
| **Destroy textures/buffers after GPU fence** | GPU may still be using resources. Use `maintain()` and queue submission ordering. |
| **Reuse command encoders per frame** | Creating a new encoder per draw call is wasteful. One encoder per render pass. |
| **`send()` staging data before next frame** | Staging belt data must be sent before the belt chunk is recycled. |

## References

- [wgpu Tutorial](https://sotrh.github.io/learn-wgpu/)
- [wgpu API Documentation](https://docs.rs/wgpu/latest/wgpu/)
- [WebGPU Specification](https://www.w3.org/TR/webgpu/)