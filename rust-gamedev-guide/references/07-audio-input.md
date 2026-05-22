# 07 — Audio & Input Systems

Status: **Expertise** | Prerequisites: 01-ecs-bevy.md

## Overview

Game feel depends on responsive input and spatial audio. Input must be buffered per-frame and processed in fixed timestep. Audio runs on separate threads via `kira` for 3D spatial positioning and `rodio` for simple playback.

## kira Spatial Audio

`kira` supports 3D audio with position-based attenuation. Sounds are positioned in world space and attenuated by distance.

```rust
use kira::{
    manager::{AudioManager, AudioManagerSettings},
    sound::static_sound::{StaticSoundData, StaticSoundSettings},
    spatial::{
        scene::{SpatialSceneHandle, SpatialSceneSettings},
        emitter::{EmitterHandle, EmitterSettings, EmitterDistances, AttenuationMode},
        listener::ListenerHandle,
    },
};

#[derive(Resource)]
struct SpatialAudioManager {
    manager: AudioManager,
    scene: SpatialSceneHandle,
    listener: ListenerHandle,
}

impl SpatialAudioManager {
    fn new() -> Result<Self, Box<dyn std::error::Error>> {
        let mut manager = AudioManager::new(AudioManagerSettings::default())?;
        let mut scene = manager.add_spatial_scene(SpatialSceneSettings::default())?;
        let listener = scene.add_listener(
            glam::Vec3::ZERO,
            glam::Quat::IDENTITY,
        )?;

        Ok(Self { manager, scene, listener })
    }

    fn play_sound_at(
        &mut self,
        sound_data: StaticSoundData,
        position: glam::Vec3,
    ) -> Result<EmitterHandle, Box<dyn std::error::Error>> {
        let settings = StaticSoundSettings::default();
        let sound = self.manager.play(sound_data)?;
        let emitter = self.scene.add_emitter(
            position,
            EmitterSettings {
                distances: EmitterDistances {
                    min_distance: 1.0,
                    max_distance: 50.0,
                },
                attenuation_mode: AttenuationMode::InverseSquare,
                ..Default::default()
            },
        )?;
        Ok(emitter)
    }

    fn update_listener(&mut self, position: glam::Vec3, orientation: glam::Quat) {
        self.listener.set_position(position);
        self.listener.set_orientation(orientation);
    }
}
```

## rodio Playback (Simple)

`rodio` is a simpler audio library for 2D playback, useful for UI sounds and background music.

```rust
use rodio::{OutputStream, OutputStreamHandle, Sink, Source};
use std::io::BufReader;

#[derive(Resource)]
struct SimpleAudio {
    stream: OutputStream,
    stream_handle: OutputStreamHandle,
}

impl SimpleAudio {
    fn new() -> Result<Self, Box<dyn std::error::Error>> {
        let (stream, stream_handle) = OutputStream::try_default()?;
        Ok(Self { stream, stream_handle })
    }

    fn play_sound(&self, sound_data: &[u8]) -> Result<Sink, Box<dyn std::error::Error>> {
        let cursor = std::io::Cursor::new(sound_data.to_vec());
        let source = rodio::Decoder::new(BufReader::new(cursor))?;
        let sink = Sink::try_new(&self.stream_handle)?;
        sink.append(source);
        Ok(sink)
    }
}
```

## Input<KeyCode>

Bevy's `Input<T>` resource tracks key/button/mouse state per frame. `just_pressed()` and `just_released()` detect transitions.

```rust
use bevy::input::ButtonInput;

fn keyboard_input_system(
    keyboard: Res<ButtonInput<KeyCode>>,
    mut query: Query<&mut Transform, With<Player>>,
    time: Res<Time>,
) {
    let Ok(mut transform) = query.get_single_mut() else { return };

    let speed = 10.0;
    let dt = time.delta_secs();

    if keyboard.pressed(KeyCode::KeyW) {
        transform.translation.z -= speed * dt;
    }
    if keyboard.pressed(KeyCode::KeyS) {
        transform.translation.z += speed * dt;
    }
    if keyboard.pressed(KeyCode::KeyA) {
        transform.translation.x -= speed * dt;
    }
    if keyboard.pressed(KeyCode::KeyD) {
        transform.translation.x += speed * dt;
    }

    if keyboard.just_pressed(KeyCode::Space) {
        info!("Jump!");
    }

    if keyboard.just_released(KeyCode::ShiftLeft) {
        info!("Sprint ended");
    }
}
```

## Input<GamepadButton>

```rust
fn gamepad_input_system(
    gamepad_buttons: Res<ButtonInput<GamepadButton>>,
    gamepad_axes: Res<Axis<GamepadAxis>>,
    gamepads: Query<&Gamepad>,
) {
    for gamepad in gamepads.iter() {
        let left_x = gamepad_axes
            .get(GamepadAxis::new(gamepad, GamepadAxisType::LeftStickX))
            .unwrap_or(0.0);

        let left_y = gamepad_axes
            .get(GamepadAxis::new(gamepad, GamepadAxisType::LeftStickY))
            .unwrap_or(0.0);

        let movement = Vec2::new(left_x, left_y);

        if gamepad_buttons.just_pressed(GamepadButton::new(
            gamepad,
            GamepadButtonType::South,
        )) {
            info!("A button pressed on gamepad {:?}", gamepad);
        }
    }
}
```

## leafwing-input-manager Action Mapping

`leafwing-input-manager` decouples input devices from game actions. "Jump" can be bound to Space, Gamepad South, or touch — the system only cares about `ActionState::just_pressed(Action::Jump)`.

```rust
use leafwing_input_manager::{
    prelude::*,
    Actionlike,
    InputManagerBundle,
};

#[derive(Actionlike, PartialEq, Eq, Hash, Clone, Copy, Debug, Reflect)]
enum PlayerAction {
    Move,
    Jump,
    Shoot,
    Interact,
    Pause,
}

fn setup_input_mapping(mut commands: Commands) {
    let input_map = InputMap::new([
        (KeyCode::Space, PlayerAction::Jump),
        (KeyCode::KeyW, PlayerAction::Move),
        (KeyCode::KeyA, PlayerAction::Move),
        (KeyCode::KeyS, PlayerAction::Move),
        (KeyCode::KeyD, PlayerAction::Move),
        (MouseButton::Left, PlayerAction::Shoot),
        (KeyCode::KeyE, PlayerAction::Interact),
        (KeyCode::Escape, PlayerAction::Pause),
        (GamepadButtonType::South, PlayerAction::Jump),
        (GamepadButtonType::RightTrigger2, PlayerAction::Shoot),
        (GamepadButtonType::Start, PlayerAction::Pause),
    ])
    .with_dual_axis_stick(
        GamepadStick::LEFT,
        VirtualDPad::wasd(
            KeyCode::KeyW,
            KeyCode::KeyS,
            KeyCode::KeyA,
            KeyCode::KeyD,
        ),
        PlayerAction::Move,
    );

    commands.spawn(InputManagerBundle::<PlayerAction> {
        action_state: ActionState::default(),
        input_map,
    });
}

fn action_driven_movement(
    query: Query<&ActionState<PlayerAction>>,
    mut transform_query: Query<&mut Transform>,
) {
    let action_state = query.single();

    if action_state.just_pressed(&PlayerAction::Jump) {
        info!("Player jumped via action mapping");
    }

    if action_state.pressed(&PlayerAction::Shoot) {
        info!("Shooting!");
    }

    let axis_pair = action_state
        .clamped_axis_pair(&PlayerAction::Move)
        .unwrap_or_default();

    for mut transform in transform_query.iter_mut() {
        transform.translation.x += axis_pair.x() * 0.1;
        transform.translation.z -= axis_pair.y() * 0.1;
    }
}
```

## Input Buffering & Fixed Timestep

Input must be collected in `Update` but processed in `FixedUpdate`. This prevents one fast frame from triggering multiple actions and ensures deterministic input handling.

```rust
#[derive(Resource, Default)]
struct InputBuffer {
    actions: Vec<BufferedAction>,
}

#[derive(Clone)]
enum BufferedAction {
    Jump,
    Shoot,
    Interact,
}

fn collect_input(
    keyboard: Res<ButtonInput<KeyCode>>,
    mut input_buffer: ResMut<InputBuffer>,
) {
    input_buffer.actions.clear();

    if keyboard.just_pressed(KeyCode::Space) {
        input_buffer.actions.push(BufferedAction::Jump);
    }
    // Note: MouseButton input requires a separate ButtonInput<MouseButton> resource.
    // This example assumes a combined input mapping for illustration.
    if mouse.just_pressed(MouseButton::Left) {
        input_buffer.actions.push(BufferedAction::Shoot);
    }
    if keyboard.just_pressed(KeyCode::KeyE) {
        input_buffer.actions.push(BufferedAction::Interact);
    }
}

fn process_input_in_fixed_update(
    mut input_buffer: ResMut<InputBuffer>,
    mut commands: Commands,
) {
    for action in input_buffer.actions.drain(..) {
        match action {
            BufferedAction::Jump => { /* apply jump */ }
            BufferedAction::Shoot => { /* spawn projectile */ }
            BufferedAction::Interact => { /* interact with nearest */ }
        }
    }
}
```

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Audio on separate thread** | Audio callbacks block. `kira` and `rodio` use internal threads. |
| **Input buffered, processed in FixedUpdate** | "One key press = many actions" is the bug. Buffer once, process once. |
| **Use action mapping, not raw keycodes** | `leafwing-input-manager` decouples devices from actions. Remappable controls. |
| **No polling input in FixedUpdate** | FixedUpdate may run 0 or N times per frame. Input state changes only in Update. |
| **`just_pressed()` for one-shot actions** | `pressed()` for continuous. `just_pressed()` fires exactly once per press. |

## References

- [kira Spatial Audio](https://docs.rs/kira/latest/kira/spatial/index.html)
- [rodio](https://docs.rs/rodio/latest/rodio/)
- [leafwing-input-manager](https://docs.rs/leafwing-input-manager/latest/leafwing_input_manager/)
- [Bevy Input System](https://bevyengine.org/learn/book/features/input/)