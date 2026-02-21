# Bevy & Rust Game Development  - Chapter 1 Reference

---

## 1. Project Setup

### 1.1 Create the Project

```bash
cargo new bevy_game
cd bevy_game
```

### 1.2 Add Bevy Dependency

In `Cargo.toml`:

```toml
[dependencies]
bevy = "0.18"
```

### 1.3 Speed Up Compile Times (Important)

Bevy is a large crate. Add these to your project for dramatically faster iteration:

**`.cargo/config.toml`** (create this file):

These flags switch the linker to `lld`, which links Bevy's large number of crates significantly faster than the default linker. On macOS, the equivalent uses LLVM's `ld64.lld`.

```toml
[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=lld"]

# macOS equivalent
[target.aarch64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/opt/homebrew/opt/llvm/bin/ld64.lld"]
```

**`Cargo.toml` profile tweaks:**

```toml
# Enable optimizations for dependencies in dev builds (huge perf boost)
[profile.dev.package."*"]
opt-level = 3

# Enable dynamic linking for Bevy (faster recompiles)
[features]
default = ["bevy/dynamic_linking"]
```

> **Tip:** Install `lld` or `mold` as your linker. On Linux: `sudo apt install lld`. This alone can cut link times from 10s+ to under 1s.

---

## 2. Core Architecture  - Thinking in Systems

Bevy is built on an **Entity Component System (ECS)** with two fundamental patterns:

| Pattern | Schedule | Runs | Use Case |
|---------|----------|------|----------|
| **Setup** | `Startup` | Once at launch | Spawn cameras, players, load assets |
| **Update** | `Update` | Every frame | Input handling, movement, animation, AI |

### Key Concepts

- **Entity:** A unique ID (like `#42`)  - holds no data itself.
- **Component:** Data attached to an entity (position, health, sprite, tags).
- **System:** A function registered with Bevy that runs at a scheduled time.
- **Resource (`Res<T>`):** Game-wide singleton data not tied to any entity (time, input state, asset server).
- **Bundle:** A group of components spawned together for convenience.
- **Plugin:** A struct implementing `Plugin` that registers systems, resources, and events  - keeps code modular.

---

## 3. Step-by-Step Implementation

### Step 1  - Minimal Window with Camera

```rust
// src/main.rs
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .run();
}

fn setup(mut commands: Commands) {
    commands.spawn(Camera2d);
}
```

**What happens:** `App::new()` creates the application → `DefaultPlugins` loads rendering, input, audio, and windowing → `setup` runs once at startup → `run()` enters Bevy's main loop (poll input → run systems → render frame → repeat).

> **Tip:** `DefaultPlugins` is configurable. You can swap out the window plugin to set title, resolution, vsync, etc:
> ```rust
> DefaultPlugins.set(WindowPlugin {
>     primary_window: Some(Window {
>         title: "My Game".into(),
>         resolution: (1280., 720.).into(),
>         ..default()
>     }),
>     ..default()
> })
> ```

---

### Step 2  - Player Component (Tag Marker)

```rust
#[derive(Component)]
struct Player;
```

- `struct Player;` is a **unit struct**  - zero-size, used purely as a tag.
- `#[derive(Component)]` is a **derive macro** that generates the boilerplate Bevy needs to store/query this type.
- Tag components let systems query for specific entities: "give me the entity with the `Player` tag."

> **Tip:** You'll use this pattern constantly. Common tags: `Player`, `Enemy`, `Bullet`, `Wall`, `NPC`.

---

### Step 3  - Spawn the Player (Text Placeholder)

```rust
fn setup(mut commands: Commands) {
    commands.spawn(Camera2d);

    commands.spawn((
        Text2d::new("@"),
        TextFont {
            font_size: 12.0,
            font: default(),
            ..default()
        },
        TextColor(Color::WHITE),
        Transform::from_translation(Vec3::ZERO),
        Player,
    ));
}
```

**What's spawned  - conceptual entity table:**

| Entity | Components |
|--------|-----------|
| `#0` | `Camera2d` |
| `#1` | `Text2d("@")`, `TextFont`, `TextColor`, `Transform`, `Player` |

- `commands.spawn(( ... ))` takes a **tuple of components** treated as a bundle.
- `Transform::from_translation(Vec3::ZERO)` places the entity at world origin `(0, 0, 0)`.
- `..default()` fills remaining struct fields with defaults  - a Rust pattern called **struct update syntax**.

---

### Step 4  - Player Movement System

```rust
fn move_player(
    input: Res<ButtonInput<KeyCode>>,               // Keyboard state
    time: Res<Time>,                                  // Frame delta
    mut player_transform: Single<&mut Transform, With<Player>>,  // The player's position
) {
    let mut direction = Vec2::ZERO;
    if input.pressed(KeyCode::ArrowLeft)  { direction.x -= 1.0; }
    if input.pressed(KeyCode::ArrowRight) { direction.x += 1.0; }
    if input.pressed(KeyCode::ArrowUp)    { direction.y += 1.0; }
    if input.pressed(KeyCode::ArrowDown)  { direction.y -= 1.0; }

    if direction != Vec2::ZERO {
        let speed = 300.0;
        let delta = direction.normalize() * speed * time.delta_secs();
        player_transform.translation.x += delta.x;
        player_transform.translation.y += delta.y;
    }
}
```

**Important details:**

- `Single<&mut Transform, With<Player>>`  - queries for **exactly one** entity matching the filter. Panics if zero or multiple match. Perfect for a single-hero game.
- `direction.normalize()`  - converts the vector to length 1 so **diagonal movement isn't ~41% faster** than cardinal movement.
- `time.delta_secs()`  - frame-time-independent movement. The player moves at the same speed regardless of FPS.
- `Res<T>` is an immutable resource borrow; `ResMut<T>` is mutable.

> **Tip:** For multiple players or more flexibility, use `Query<&mut Transform, With<Player>>` instead of `Single`. Iterate with `for mut transform in &mut query { ... }`.

### Step 5  - Register the Movement System

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .add_systems(Update, move_player)  // Runs every frame
        .run();
}
```

---

## 4. Adding Sprite Graphics

### 4.1 Spritesheet Source

Use the [Universal LPC Spritesheet Generator](https://liberatedpixelcup.github.io/Universal-LPC-Spritesheet-Character-Generator/) to create character spritesheets. Export as PNG. The sheet uses a grid of **64×64 tiles** with 9 walk frames per direction row.

### 4.2 Asset Directory

Create `src/assets/` and place your spritesheet there (e.g., `male_spritesheet.png`).

> **Tip:** The standard Bevy convention is an `assets/` folder at the project root, not `src/assets/`. The tutorial overrides this via `AssetPlugin`. For new projects, prefer the default `assets/` root to avoid configuration.

### 4.3 Refactor  - Separate into Modules

**`src/main.rs`:**

```rust
use bevy::prelude::*;

mod player;
use crate::player::PlayerPlugin;

fn main() {
    App::new()
        .insert_resource(ClearColor(Color::WHITE))
        .add_plugins(
            DefaultPlugins.set(AssetPlugin {
                file_path: "src/assets".into(),
                ..default()
            }),
        )
        .add_systems(Startup, setup_camera)
        .add_plugins(PlayerPlugin)
        .run();
}

fn setup_camera(mut commands: Commands) {
    commands.spawn(Camera2d);
}
```

- `insert_resource(ClearColor(Color::WHITE))`  - global background color.
- `mod player;` pulls in `src/player.rs` as a module.

---

### 4.4 The Player Module (`src/player.rs`)

#### Constants

`TILE_SIZE` is 64 to match the LPC spritesheet's 64×64 pixel grid. `WALK_FRAMES` is 9 because the LPC convention uses 9 frames per walk cycle row. `ANIM_DT` at 0.1 seconds per frame gives roughly 10fps animation playback, which looks natural for pixel art walking.

```rust
use bevy::prelude::*;

const TILE_SIZE: u32 = 64;
const WALK_FRAMES: usize = 9;
const MOVE_SPEED: f32 = 140.0;
const ANIM_DT: f32 = 0.1;      // ~10 FPS animation
```

#### Components

```rust
#[derive(Component)]
struct Player;

#[derive(Component, Debug, Clone, Copy, PartialEq, Eq)]
enum Facing {
    Up,
    Left,
    Down,
    Right,
}

#[derive(Component, Deref, DerefMut)]
struct AnimationTimer(Timer);

#[derive(Component)]
struct AnimationState {
    facing: Facing,
    moving: bool,
    was_moving: bool,
}
```

**Key Rust concepts:**

- **`enum` vs `struct`:** Use `enum` when you pick *one option* from a set (direction, weapon type, game state). Use `struct` when you need *multiple fields together* (position with x/y, stats with health/mana).
- **`Deref` / `DerefMut`:** Lets `AnimationTimer` transparently behave like the inner `Timer`  - you can call `.tick()`, `.reset()`, etc. directly on it.
- **`Clone` + `Copy`:** Rust *moves* values by default (original becomes invalid). `Copy` opts into cheap bitwise duplication for small types. `PartialEq` / `Eq` enable `==` comparisons.

#### Spawn System

```rust
fn spawn_player(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    mut atlas_layouts: ResMut<Assets<TextureAtlasLayout>>,
) {
    let texture = asset_server.load("male_spritesheet.png");
    let layout = atlas_layouts.add(TextureAtlasLayout::from_grid(
        UVec2::splat(TILE_SIZE),
        WALK_FRAMES as u32,
        12,
        None,
        None,
    ));

    let facing = Facing::Down;
    let start_index = atlas_index_for(facing, 0);

    commands.spawn((
        Sprite::from_atlas_image(
            texture,
            TextureAtlas { layout, index: start_index },
        ),
        Transform::from_translation(Vec3::ZERO),
        Player,
        AnimationState { facing, moving: false, was_moving: false },
        AnimationTimer(Timer::from_seconds(ANIM_DT, TimerMode::Repeating)),
    ));
}
```

**Important details:**

- `AssetServer` performs **lazy/async loading**  - `load()` returns a handle immediately. The actual file loads in the background. Multiple entities can share the same handle without duplicating memory.
- `TextureAtlasLayout::from_grid(...)` tells Bevy how to slice the spritesheet into individual frames.
- `ResMut<Assets<TextureAtlasLayout>>`  - mutable access to Bevy's asset storage to register our layout.
- `TimerMode::Repeating`  - the timer automatically resets and fires again after each interval.

#### Movement System (Updated for Animation)

```rust
fn move_player(
    input: Res<ButtonInput<KeyCode>>,
    time: Res<Time>,
    mut player: Query<(&mut Transform, &mut AnimationState), With<Player>>,
) {
    let Ok((mut transform, mut anim)) = player.single_mut() else {
        return;
    };

    let mut direction = Vec2::ZERO;
    if input.pressed(KeyCode::ArrowLeft)  { direction.x -= 1.0; }
    if input.pressed(KeyCode::ArrowRight) { direction.x += 1.0; }
    if input.pressed(KeyCode::ArrowUp)    { direction.y += 1.0; }
    if input.pressed(KeyCode::ArrowDown)  { direction.y -= 1.0; }

    if direction != Vec2::ZERO {
        let delta = direction.normalize() * MOVE_SPEED * time.delta_secs();
        transform.translation.x += delta.x;
        transform.translation.y += delta.y;
        anim.moving = true;

        // Dominant axis determines facing
        if direction.x.abs() > direction.y.abs() {
            anim.facing = if direction.x > 0.0 { Facing::Right } else { Facing::Left };
        } else {
            anim.facing = if direction.y > 0.0 { Facing::Up } else { Facing::Down };
        }
    } else {
        anim.moving = false;
    }
}
```

> **Tip:** The `let Ok(...) = ... else { return; }` pattern is **let-else** syntax (Rust 1.65+). It's the cleanest way to early-return on query failure.

#### Animation System

```rust
fn animate_player(
    time: Res<Time>,
    mut query: Query<(&mut AnimationState, &mut AnimationTimer, &mut Sprite), With<Player>>,
) {
    let Ok((mut anim, mut timer, mut sprite)) = query.single_mut() else {
        return;
    };

    let atlas = match sprite.texture_atlas.as_mut() {
        Some(a) => a,
        None => return,
    };

    let target_row = row_zero_based(anim.facing);
    let mut current_col = atlas.index % WALK_FRAMES;
    let mut current_row = atlas.index / WALK_FRAMES;

    // Snap to correct row on direction change
    if current_row != target_row {
        atlas.index = row_start_index(anim.facing);
        current_col = 0;
        current_row = target_row;
        timer.reset();
    }

    let just_started = anim.moving && !anim.was_moving;
    let just_stopped = !anim.moving && anim.was_moving;

    if anim.moving {
        if just_started {
            let row_start = row_start_index(anim.facing);
            let next_col = (current_col + 1) % WALK_FRAMES;
            atlas.index = row_start + next_col;
            timer.reset();
        } else {
            timer.tick(time.delta());
            if timer.just_finished() {
                let row_start = row_start_index(anim.facing);
                let next_col = (current_col + 1) % WALK_FRAMES;
                atlas.index = row_start + next_col;
            }
        }
    } else if just_stopped {
        timer.reset();
    }

    anim.was_moving = anim.moving;
}
```

**Animation logic breakdown:**

1. Detect if the facing direction changed → snap to the new row's first frame.
2. If the player **just started** moving → immediately advance one frame for responsive feedback.
3. If **continuously moving** → advance frames on timer ticks (~10 FPS).
4. If **just stopped** → freeze on current frame, reset timer.

#### Helper Functions

```rust
fn row_start_index(facing: Facing) -> usize {
    row_zero_based(facing) * WALK_FRAMES
}

fn atlas_index_for(facing: Facing, frame_in_row: usize) -> usize {
    row_start_index(facing) + frame_in_row.min(WALK_FRAMES - 1)
}

fn row_zero_based(facing: Facing) -> usize {
    match facing {
        Facing::Up    => 8,
        Facing::Left  => 9,
        Facing::Down  => 10,
        Facing::Right => 11,
    }
}
```

> **Note:** These row indices (8–11) are specific to the LPC spritesheet layout where walk animations are in rows 8–11. Different spritesheets will have different layouts.

#### Player Plugin

```rust
pub struct PlayerPlugin;

impl Plugin for PlayerPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(Startup, spawn_player)
            .add_systems(Update, (move_player, animate_player));
    }
}
```

- A **trait** is a contract (interface). `Plugin` requires a `build` method.
- `(move_player, animate_player)`  - passing a tuple registers multiple systems in the same schedule. They run in parallel by default unless they have conflicting data access.

> **Tip:** Use `.chain()` to enforce ordering: `.add_systems(Update, (move_player, animate_player).chain())` guarantees `move_player` runs before `animate_player` every frame.

---

## 5. Rust Concepts Quick Reference

| Concept | What It Means | When to Use |
|---------|--------------|-------------|
| `mut` | Mutable binding  - value can be changed | Anytime you need to modify a variable |
| `struct` | Group of named fields | Position, stats, config data |
| `enum` | One-of-many variants | Directions, states, weapon types |
| `derive` macro | Auto-generate trait implementations | `Component`, `Debug`, `Clone`, `Copy`, `PartialEq` |
| `Deref` / `DerefMut` | Transparent wrapper  - inner type's methods available directly | Newtype pattern (e.g., `AnimationTimer(Timer)`) |
| Ownership & Move | Values have one owner; assignment transfers ownership | Default Rust behavior; use `Copy`/`Clone` to opt out |
| `Result<T, E>` | Success (`Ok`) or failure (`Err`) | Fallible operations, query results |
| `Option<T>` | Value present (`Some`) or absent (`None`) | Nullable fields, optional returns |
| `let ... else` | Destructure or early-return | Clean guard clauses |
| `match` | Exhaustive pattern matching | Enums, Options, Results |
| `..default()` | Fill remaining fields with defaults | Struct initialization shorthand |

---

## 6. Bevy Memory & Performance Notes

Bevy's ECS stores components in **archetypes**  - tightly packed arrays of the same component types. This means:

- `Vec2` in Rust = exactly **8 bytes** (two `f32`). In JS/Python, the same concept uses ~48+ bytes due to runtime metadata.
- With 1,000 entities: Rust uses ~8 KB vs ~48 KB+ in dynamic languages.
- Tight packing = better **CPU cache utilization** = faster frame rates.
- Type checking happens at **compile time**, not runtime  - zero overhead.

---

## 7. Recommended Crates for Expanding This Project

### Physics & Collision

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`avian2d`](https://crates.io/crates/avian2d) | 2D physics engine for Bevy | The modern standard; rigid bodies, colliders, joints. Replaces `bevy_rapier2d` as the go-to. |
| [`bevy_rapier2d`](https://crates.io/crates/bevy_rapier2d) | Rapier physics integration | Battle-tested, still widely used. Slightly more complex API than Avian. |

### Tilemap & World Building

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_ecs_tilemap`](https://crates.io/crates/bevy_ecs_tilemap) | Performant ECS-native tilemaps | Great for large worlds; each tile is an entity. |
| [`bevy_ecs_ldtk`](https://crates.io/crates/bevy_ecs_ldtk) | LDtk level editor integration | Visual level design with auto-spawning of entities. |
| [`bevy_tiled`](https://crates.io/crates/bevy_tiled) | Tiled map editor integration | If you prefer Tiled over LDtk. |

### Animation & VFX

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_spritesheet_animation`](https://crates.io/crates/bevy_spritesheet_animation) | Declarative sprite animations | Cleaner API than manual atlas cycling; supports animation clips and transitions. |
| [`bevy_hanabi`](https://crates.io/crates/bevy_hanabi) | GPU particle effects | Dust trails, explosions, magic effects. |
| [`bevy_tweening`](https://crates.io/crates/bevy_tweening) | Tweening / easing animations | Smooth UI transitions, camera movements, entity interpolation. |

### Input

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`leafwing-input-manager`](https://crates.io/crates/leafwing-input-manager) | Action-based input mapping | Maps physical keys to game actions. Supports gamepads, rebinding, chords. Essential for any serious game. |

### Camera

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_pancam`](https://crates.io/crates/bevy_pancam) | Pan/zoom 2D camera | Quick setup for scrollable worlds. |
| [`bevy_pixel_camera`](https://crates.io/crates/bevy_pixel_camera) | Pixel-perfect rendering | Prevents sub-pixel blurriness in pixel art games. |

### UI & Debug

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy-inspector-egui`](https://crates.io/crates/bevy-inspector-egui) | Runtime entity/resource inspector | Invaluable for debugging  - inspect and modify components live. |
| [`bevy_egui`](https://crates.io/crates/bevy_egui) | egui integration | Quick debug UIs, settings panels, dev tools. |
| [`iyes_perf_ui`](https://crates.io/crates/iyes_perf_ui) | FPS/diagnostics overlay | Frame time, entity count, system timing. |

### Audio

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_kira_audio`](https://crates.io/crates/bevy_kira_audio) | Advanced audio with Kira backend | Spatial audio, crossfading, audio buses. Much richer than Bevy's built-in audio. |

### State Management & AI

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`big-brain`](https://crates.io/crates/big-brain) | Utility AI for NPCs | Score-based decision making; great for enemy behaviors. |
| [`bonsai-bt`](https://crates.io/crates/bonsai-bt) | Behavior trees | Classic game AI pattern for complex NPC logic. |
| [`seldom_state`](https://crates.io/crates/seldom_state) | State machine for entities | Clean state transitions for player/enemy states. |

### Networking

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`lightyear`](https://crates.io/crates/lightyear) | Client/server netcode for Bevy | Prediction, interpolation, lag compensation. The most complete Bevy networking solution. |

### Saving & Serialization

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_save`](https://crates.io/crates/bevy_save) | Save/load game state | Snapshots, rollbacks, scene serialization. |

---

## 8. Tips for Further Improvement

### Immediate Next Steps

1. **Add WASD support alongside arrow keys**  - check for both `KeyCode::KeyW` and `KeyCode::ArrowUp` in your input system for better ergonomics.
2. **Implement camera follow**  - instead of a static camera, make it track the player's `Transform` with optional smoothing via `lerp`.
3. **Add a movement speed component**  - replace the hardcoded `MOVE_SPEED` constant with a `MoveSpeed(f32)` component on the player entity. This lets different entities have different speeds.
4. **Sprint mechanic**  - check `input.pressed(KeyCode::ShiftLeft)` and multiply speed by a factor.

### Architecture Tips

5. **Use `States` for game flow**  - Bevy's built-in `States` enum (e.g., `MainMenu`, `InGame`, `Paused`) controls which systems run when. Add systems with `.run_if(in_state(GameState::InGame))`.
6. **Use Events for communication**  - instead of checking flags every frame, use `EventWriter<T>` and `EventReader<T>` for things like "player attacked", "item picked up", "enemy died".
7. **Organize by feature, not file type**  - instead of `components.rs`, `systems.rs`, prefer `player.rs`, `enemy.rs`, `combat.rs` each containing their own components + systems + plugin.
8. **System ordering matters**  - if your animation looks jittery, chain your systems: `(move_player, animate_player).chain()` or use explicit ordering with `.before()` / `.after()`.

### Performance Tips

9. **Use `SpriteBundle` transforms wisely**  - avoid changing `Transform.scale` for sprite resizing; use the sprite's `custom_size` field instead.
10. **Batch spawn with `commands.spawn_batch()`**  - when spawning many entities (enemies, particles), batch spawning is significantly faster.
11. **Run Bevy's diagnostics**  - add `LogDiagnosticsPlugin` and `FrameTimeDiagnosticsPlugin` to track frame times during development.

### Debug Workflow

12. **Hot reloading assets**  - Bevy supports watching the asset directory for changes. Add `AssetPlugin { watch_for_changes: true, .. }` during development to see sprite changes without restarting.
13. **Use `info!()`, `warn!()`, `error!()`**  - Bevy re-exports `tracing` macros. Prefer these over `println!()` for structured, filterable logs.

---

## 9. Complete File Structure

```
bevy_game/
├── Cargo.toml
├── .cargo/
│   └── config.toml          # Linker optimizations
├── src/
│   ├── main.rs              # App setup, camera, plugin registration
│   ├── player.rs            # Player components, systems, plugin
│   └── assets/
│       └── male_spritesheet.png
```

---

## 10. Full Schedule Reference

```
Startup          → Runs once before first frame
PreUpdate        → Bevy internals (input collection, events)
Update           → Your game logic (movement, AI, gameplay)
PostUpdate       → Bevy internals (transform propagation, rendering prep)
FixedUpdate      → Fixed timestep (physics, deterministic logic)
```

> **Tip:** Use `FixedUpdate` for physics and anything that needs deterministic behavior. Use `Update` for input handling and visual updates.

---

## API Quick Reference

APIs introduced in this chapter. See the [API Glossary](./api-glossary.md) for all APIs across the book.

| API | Category | Description |
|-----|----------|-------------|
| [`App`](https://docs.rs/bevy/0.18.0/bevy/app/struct.App.html) | Core ECS | Application builder and entry point |
| [`Commands`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Commands.html) | Core ECS | Deferred command queue for spawning/despawning entities |
| [`Component`](https://docs.rs/bevy/0.18.0/bevy/ecs/component/trait.Component.html) | Core ECS | Derive macro to mark a type as attachable to entities |
| [`Plugin`](https://docs.rs/bevy/0.18.0/bevy/app/trait.Plugin.html) | Core ECS | Trait for modular code organization |
| [`Query<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Query.html) | Core ECS | System parameter for reading/writing entity components |
| [`Res<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Res.html) | Core ECS | Immutable access to a global resource |
| [`ResMut<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.ResMut.html) | Core ECS | Mutable access to a global resource |
| [`Resource`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/trait.Resource.html) | Core ECS | Derive macro for global singleton data |
| [`Single<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Single.html) | Core ECS | Query that expects exactly one matching entity |
| [`Startup`](https://docs.rs/bevy/0.18.0/bevy/app/struct.Startup.html) | Core ECS | Schedule that runs systems once at launch |
| [`Update`](https://docs.rs/bevy/0.18.0/bevy/app/struct.Update.html) | Core ECS | Schedule that runs systems every frame |
| [`With<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/query/struct.With.html) | Core ECS | Query filter requiring a component's presence |
| [`Camera2d`](https://docs.rs/bevy/0.18.0/bevy/core_pipeline/core_2d/struct.Camera2d.html) | Rendering | Marker component for 2D camera entities |
| [`Color`](https://docs.rs/bevy/0.18.0/bevy/color/enum.Color.html) | Rendering | Color representation (multiple color spaces) |
| [`Sprite`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.Sprite.html) | Rendering | 2D image rendering component |
| [`Text2d`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.Text2d.html) | Rendering | 2D text rendering component |
| [`TextColor`](https://docs.rs/bevy/0.18.0/bevy/text/struct.TextColor.html) | Rendering | Text color component |
| [`TextFont`](https://docs.rs/bevy/0.18.0/bevy/text/struct.TextFont.html) | Rendering | Font configuration for text rendering |
| [`TextureAtlas`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.TextureAtlas.html) | Rendering | Sprite atlas frame selection |
| [`TextureAtlasLayout`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.TextureAtlasLayout.html) | Rendering | Defines how a spritesheet is sliced |
| [`Transform`](https://docs.rs/bevy/0.18.0/bevy/transform/components/struct.Transform.html) | Math | Position, rotation, and scale of an entity |
| [`UVec2`](https://docs.rs/bevy/0.18.0/bevy/math/struct.UVec2.html) | Math | Unsigned integer 2D vector |
| [`Vec2`](https://docs.rs/bevy/0.18.0/bevy/math/struct.Vec2.html) | Math | 2D floating-point vector |
| [`Vec3`](https://docs.rs/bevy/0.18.0/bevy/math/struct.Vec3.html) | Math | 3D floating-point vector |
| [`AssetServer`](https://docs.rs/bevy/0.18.0/bevy/asset/struct.AssetServer.html) | Assets | Loads and manages game assets |
| [`Assets<T>`](https://docs.rs/bevy/0.18.0/bevy/asset/struct.Assets.html) | Assets | Typed storage for loaded assets |
| [`Handle<T>`](https://docs.rs/bevy/0.18.0/bevy/asset/enum.Handle.html) | Assets | Reference-counted pointer to a loaded asset |
| [`ButtonInput<KeyCode>`](https://docs.rs/bevy/0.18.0/bevy/input/struct.ButtonInput.html) | Input | Keyboard state resource |
| [`KeyCode`](https://docs.rs/bevy/0.18.0/bevy/input/keyboard/enum.KeyCode.html) | Input | Identifies a physical keyboard key |
| [`Time`](https://docs.rs/bevy/0.18.0/bevy/time/struct.Time.html) | Time | Frame timing and delta time resource |
| [`Timer`](https://docs.rs/bevy/0.18.0/bevy/time/struct.Timer.html) | Time | Countdown or repeating timer |
| [`TimerMode`](https://docs.rs/bevy/0.18.0/bevy/time/enum.TimerMode.html) | Time | Once or Repeating timer behavior |
| [`AssetPlugin`](https://docs.rs/bevy/0.18.0/bevy/asset/struct.AssetPlugin.html) | Plugins | Configures asset loading paths |
| [`ClearColor`](https://docs.rs/bevy/0.18.0/bevy/camera/struct.ClearColor.html) | Plugins | Global background color resource |
| [`DefaultPlugins`](https://docs.rs/bevy/0.18.0/bevy/struct.DefaultPlugins.html) | Plugins | Standard plugin group |
| [`ImagePlugin`](https://docs.rs/bevy/0.18.0/bevy/render/texture/struct.ImagePlugin.html) | Plugins | Configures image loading defaults |
| [`Window`](https://docs.rs/bevy/0.18.0/bevy/window/struct.Window.html) | Plugins | Window configuration |
| [`WindowPlugin`](https://docs.rs/bevy/0.18.0/bevy/window/struct.WindowPlugin.html) | Plugins | Configures window creation |
| [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html) | Rust std | Deep-copy capability |
| [`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html) | Rust std | Cheap bitwise duplication |
| [`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html) | Rust std | Debug formatting (`{:?}`) |
| [`Default`](https://doc.rust-lang.org/std/default/trait.Default.html) | Rust std | Default value generation |
| [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html) | Rust std | Transparent wrapper delegation |
| [`DerefMut`](https://doc.rust-lang.org/std/ops/trait.DerefMut.html) | Rust std | Mutable wrapper delegation |
| [`Eq`](https://doc.rust-lang.org/std/cmp/trait.Eq.html) | Rust std | Total equality |
| [`Option<T>`](https://doc.rust-lang.org/std/option/enum.Option.html) | Rust std | Optional value (`Some` or `None`) |
| [`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) | Rust std | Equality comparison (`==`) |
| [`Result<T, E>`](https://doc.rust-lang.org/std/result/enum.Result.html) | Rust std | Success or failure return type |
| [`Vec<T>`](https://doc.rust-lang.org/std/vec/struct.Vec.html) | Rust std | Growable heap-allocated array |