# 06 — Asset Pipeline

Status: **Core** | Prerequisites: 01-ecs-bevy.md, 03-gpu-wgpu.md

## Overview

Games have thousands of assets: textures, meshes, audio, fonts, shaders. Loading must be async, cache-aware, and hot-reloadable. Bevy's `AssetServer` handles loading, `Handle<T>` manages lifetime, and the asset system supports in-place hot-reloading.

## AssetServer Async Loading

`AssetServer::load()` returns a `Handle<T>` immediately. The asset loads asynchronously in the background. Systems can query `Handle<T>` without waiting.

```rust
use bevy::prelude::*;

#[derive(Resource)]
struct GameAssets {
    player_mesh: Handle<Mesh>,
    player_material: Handle<StandardMaterial>,
    enemy_mesh: Handle<Mesh>,
    background_texture: Handle<Image>,
    ui_font: Handle<Font>,
    explosion_scene: Handle<Scene>,
}

fn load_game_assets(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.insert_resource(GameAssets {
        player_mesh: asset_server.load("models/player.glb#Mesh0/Primitive0"),
        player_material: asset_server.load("materials/player.mat.ron"),
        enemy_mesh: asset_server.load("models/enemy.glb#Mesh0/Primitive0"),
        background_texture: asset_server.load("textures/background.png"),
        ui_font: asset_server.load("fonts/FiraSans-Bold.ttf"),
        explosion_scene: asset_server.load("scenes/explosion.glb#Scene0"),
    });
}
```

## Handle<T> Lifetimes

`Handle<T>` is a shared reference-counted pointer to an asset. Multiple entities can share the same handle. When all handles are dropped, the asset is eligible for deallocation.

```rust
fn spawn_enemies(
    mut commands: Commands,
    game_assets: Res<GameAssets>,
) {
    for i in 0..100 {
        commands.spawn((
            Enemy,
            SceneRoot(game_assets.enemy_mesh.clone()),
            Transform::from_xyz(i as f32 * 3.0, 0.0, 0.0),
        ));
    }
}
```

### Handle Strengths

| Type | Behavior |
|------|----------|
| `Handle<T>` | Strong reference. Keeps asset loaded. |
| `WeakHandle<T>` | Weak reference. Does not prevent unloading. |
| `UntypedHandle` | Erased type handle. Used for generic asset storage. |

```rust
fn weak_handle_example(mut commands: Commands, asset_server: Res<AssetServer>) {
    let strong: Handle<Image> = asset_server.load("textures/icon.png");
    let weak: WeakHandle<Image> = strong.clone_weak();

    commands.spawn(UiIcon {
        image: strong,
        fallback: weak,
    });
}
```

## Hot-Reloading

Enable file watching in development builds. Changed assets are reloaded and all handles are updated in-place — entities referencing the asset see the new version immediately.

```rust
fn configure_asset_hot_reload(asset_server: Res<AssetServer>) {
    #[cfg(debug_assertions)]
    {
        asset_server.watch_for_changes().unwrap();
    }
}
```

### Custom Asset Loader

```rust
use bevy::asset::{AssetLoader, LoadContext, AsyncReadExt};

#[derive(Asset, TypePath, Debug)]
struct LevelData {
    tiles: Vec<Vec<u32>>,
    spawn_points: Vec<Vec2>,
}

struct LevelAssetLoader;

impl AssetLoader for LevelAssetLoader {
    type Asset = LevelData;
    type Settings = ();
    type Error = std::io::Error;

    fn load<'a>(
        &'a self,
        reader: &'a mut bevy::asset::io::Reader,
        _settings: &'a Self::Settings,
        load_context: &'a mut LoadContext,
    ) -> bevy::utils::BoxedFuture<'a, Result<Self::Asset, Self::Error>> {
        Box::pin(async move {
            let mut bytes = Vec::new();
            reader.read_to_end(&mut bytes).await?;
            let level: LevelData = ron::de::from_bytes(&bytes)
                .map_err(|e| std::io::Error::new(std::io::ErrorKind::Other, e))?;
            Ok(level)
        })
    }

    fn extensions(&self) -> &[&str] {
        &["level.ron"]
    }
}

fn register_level_loader(app: &mut App) {
    app.init_asset::<LevelData>()
        .register_asset_loader(LevelAssetLoader);
}
```

## Texture Compression

BC7 (desktop) and ASTC (mobile) formats dramatically reduce GPU memory usage and bandwidth.

### BC7 Compression Setting

```rust
use bevy::render::texture::{
    ImageSampler, ImageSamplerDescriptor,
    CompressedImageFormats,
};

fn configure_texture_compression(app: &mut App) {
    app.insert_resource(CompressedImageFormats::all());
}

fn configure_texture_sampler(
    mut images: ResMut<Assets<Image>>,
    game_assets: Res<GameAssets>,
) {
    if let Some(image) = images.get_mut(&game_assets.background_texture) {
        image.sampler = ImageSampler::Descriptor(ImageSamplerDescriptor {
            mag_filter: bevy::render::render_resource::FilterMode::Linear,
            min_filter: bevy::render::render_resource::FilterMode::Linear,
            mipmap_filter: bevy::render::render_resource::FilterMode::Linear,
            anisotropy_clamp: 16,
            ..default()
        });
    }
}
```

## Asset Preprocessing

Precompile assets offline to avoid runtime cost. Bevy supports `.meta` files for controlling import settings.

```rust
fn configure_asset_processing(app: &mut App) {
    app.add_plugins(bevy::asset::AssetPlugin {
        mode: bevy::asset::AssetMode::Processed,
        file_path: "assets".to_string(),
        processed_file_path: "imported_assets/Default".to_string(),
        watch_for_changes_override: None,
        ..default()
    });
}
```

### Structured Asset Organization

```
assets/
├── models/
│   ├── player.glb
│   ├── enemy.glb
│   └── props/
│       ├── barrel.glb
│       └── crate.glb
├── textures/
│   ├── environment/
│   │   ├── skybox.hdr
│   │   └── ground_albedo.png
│   ├── characters/
│   │   ├── player_albedo.png
│   │   └── player_normal.png
│   └── ui/
│       ├── button.png
│       └── health_bar.png
├── audio/
│   ├── music/
│   │   └── background.ogg
│   └── sfx/
│       ├── explosion.wav
│       └── pickup.wav
├── fonts/
│   └── FiraSans-Bold.ttf
├── shaders/
│   ├── pbr.wgsl
│   └── particle_compute.wgsl
└── scenes/
    └── level_01.scn.ron
```

## Asset Lifecycle Management

```rust
fn unload_unused_assets(
    mut assets: ResMut<Assets<Image>>,
    used_handles: Query<&Handle<Image>>,
) {
    let mut in_use: HashSet<_> = used_handles.iter()
        .map(|h| h.id())
        .collect();

    let to_remove: Vec<_> = assets.iter()
        .filter(|(id, _)| !in_use.contains(id))
        .map(|(id, _)| id)
        .collect();

    for id in to_remove {
        assets.remove(id);
    }
}
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Never synchronously load on main thread** | Blocks frame. Use `AssetServer::load()` + `Handle<T>`. |
| **`Handle<T>` is cheap to clone** | Reference-counted. Clone freely for shared assets. |
| **Enable `watch_for_changes()` in dev** | Iteration speed. No restart needed for asset tweaks. |
| **Compress textures offline** | BC7/ASTC reduces VRAM by 4-8x. Do not ship uncompressed textures. |
| **Organize assets by type, not by level** | "textures/environment/skybox" is discoverable. "level1/skybox.png" is not. |
| **Use `AssetMode::Processed` in release** | Pre-processed assets skip import logic, reducing load times. |

## References

- [Bevy Asset Cookbook](https://bevyengine.org/learn/book/assets/)
- [BC7 Texture Compression](https://learn.microsoft.com/en-us/windows/win32/direct3d11/bc7-format)
- [ASTC Texture Compression](https://arm-software.github.io/opengl-es-sdk-for-android/astc_texture_compression.html)