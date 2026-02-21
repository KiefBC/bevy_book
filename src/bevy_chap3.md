# Bevy & Rust Game Development  - Chapter 3 Reference

---

## 1. Chapter Overview

This chapter replaces Chapter 1's hardcoded player with a **data-driven character system**. All character attributes (speed, health, animations, spritesheets) are defined in a single `.ron` config file. The code is generic  - one animation system, one movement system, one spawn system  - that works with any character.

### What Gets Built

- External `characters.ron` file defining 6 characters with unique stats and animations
- Generic animation engine supporting Walk, Run, and Jump with directional spritesheets
- Runtime character switching via number keys (1–6)
- Async asset loading with a self-terminating initialization pattern
- Sprint mechanic (hold Shift) with per-character speed multipliers

### Why Data-Driven Design

| Hardcoded (Ch1) | Data-Driven (Ch3) |
|-----------------|-------------------|
| Constants per character (`WARRIOR_SPEED`, `MAGE_SPEED`) | Single `CharacterEntry` struct read from `.ron` |
| Duplicate functions per character (`animate_warrior`, `animate_mage`) | One `animate_characters` system for all |
| Code change required to add/tweak characters | Edit `characters.ron`  - no recompile needed |
| Bug fix = update N functions | Bug fix = update 1 function |
| No runtime switching | Swap character data at runtime instantly |

---

## 2. Project Setup

### New Dependencies

```toml
[dependencies]
bevy = "0.18"
bevy_procedural_tilemaps = "0.2.0"
bevy_common_assets = { version = "0.15.0-rc.1", features = ["ron"] }
serde = { version = "1.0", features = ["derive"] }
```

- **`bevy_common_assets`**  - Asset loader plugins for common formats. The `ron` feature enables loading `.ron` files as Bevy assets.
- **`serde`**  - Serialization/deserialization with `#[derive(Serialize, Deserialize)]` for structs that map to the `.ron` file.

### File Structure

```
src/
├── main.rs
├── map/                          # From Chapter 2
│   └── (unchanged)
└── characters/
    ├── mod.rs                    # CharactersPlugin definition
    ├── config.rs                 # Data types: AnimationType, CharacterEntry, CharactersList
    ├── animation.rs              # Facing, AnimationClip, AnimationController, animate system
    ├── movement.rs               # Player marker, input reading, move_player, jump handling
    └── spawn.rs                  # Spawn, async init, character switching
```

### Asset Structure

```
src/assets/
├── characters/
│   └── characters.ron            # All character definitions
├── male_spritesheet.png
├── female_spritesheet.png
├── crimson_count_spritesheet.png
├── graveyard_reaper_spritesheet.png
├── lantern_warden_spritesheet.png
├── starlit_oracle_spritesheet.png
└── tile_layers/
    └── tilemap.png               # From Chapter 2
```

> **Important:** Delete `src/player.rs` from Chapter 1 and remove `mod player;` / `PlayerPlugin` from `main.rs`. The new `characters` module replaces it entirely.

---

## 3. The RON Configuration File

### What Is RON?

RON (Rusty Object Notation) is a human-readable data format designed for Rust. Compared to JSON:

| Feature | JSON | RON |
|---------|------|-----|
| Key quotes | Required | Optional for identifiers |
| Comments | Not supported | `//` and `/* */` supported |
| Trailing commas | Syntax error | Allowed |
| Rust types | Limited to JS types | Native tuples, structs, enums |
| Enum variants | Encoded as strings | First-class (`Walk`, `Run`, `Jump`) |

### Character Schema

Every entry in `characters.ron` follows this structure:

```ron
(
    characters: [
        (
            name: "Warrior",
            max_health: 150.0,
            base_move_speed: 140.0,
            run_speed_multiplier: 1.8,
            texture_path: "male_spritesheet.png",
            tile_size: 64,
            atlas_columns: 9,
            animations: {
                Walk: (
                    start_row: 8,       // Row in the spritesheet (0-indexed from top)
                    frame_count: 9,     // Number of frames in the animation
                    frame_time: 0.1,    // Seconds per frame
                    directional: true,  // true = 4 rows (Up/Left/Down/Right)
                ),
                Run: (
                    start_row: 8,
                    frame_count: 9,
                    frame_time: 0.065,
                    directional: true,
                ),
                Jump: (
                    start_row: 5,
                    frame_count: 7,
                    frame_time: 0.08,
                    directional: false, // Same row regardless of facing
                ),
            },
        ),
        // ... more characters
    ]
)
```

**Key fields:**
- `directional: true`  - spritesheet has 4 consecutive rows: Up (start_row+0), Left (+1), Down (+2), Right (+3).
- `directional: false`  - single row used for all directions (e.g., Jump plays the same regardless of facing).
- `atlas_columns`  - number of sprite columns in the sheet. Used to calculate frame index: `row * columns + frame`.

---

## 4. Architecture  - System Flow

```
characters.ron  ──load──►  CharactersList (asset)
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
              spawn_player   initialize   switch_character
              (Startup)      _player      (Update, keys 1-9)
                             (Update,
                              self-terminating)
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
              move_player   update_jump  animate_characters
              (input →      _state       (timer tick →
               Transform,   (jump done   frame advance)
               AnimCtrl)    → Walk)
                                           │
                                           ▼
                                    update_animation_flags
                                    (was_moving = is_moving)
```

### System Execution Order (all in Update)

1. `initialize_player_character`  - adds components once asset is loaded (self-terminates)
2. `switch_character`  - swaps character data on digit key press
3. `move_player`  - reads input, updates Transform and AnimationController
4. `update_jump_state`  - detects jump animation completion, resets to Walk
5. `animate_characters`  - ticks timer, advances sprite frame index
6. `update_animation_flags`  - stores current frame's state for next-frame change detection

---

## 5. Data Types (`config.rs`)

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum AnimationType {
    Walk,
    Run,
    Jump,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AnimationDefinition {
    pub start_row: usize,       // First row in the spritesheet
    pub frame_count: usize,     // Number of frames
    pub frame_time: f32,        // Seconds per frame
    pub directional: bool,      // 4-row directional or single-row
}

#[derive(Component, Asset, TypePath, Debug, Clone, Serialize, Deserialize)]
pub struct CharacterEntry {
    pub name: String,
    pub max_health: f32,
    pub base_move_speed: f32,
    pub run_speed_multiplier: f32,
    pub texture_path: String,
    pub tile_size: u32,
    pub atlas_columns: usize,
    pub animations: HashMap<AnimationType, AnimationDefinition>,
}

#[derive(Asset, TypePath, Debug, Clone, Serialize, Deserialize)]
pub struct CharactersList {
    pub characters: Vec<CharacterEntry>,
}
```

`CharacterEntry` is both an `Asset` (loadable from disk) and a `Component` (attachable to entities). This dual nature means the loaded data is directly queryable in ECS systems.

### `calculate_max_animation_row()`

Inspects all animation definitions to determine the total rows needed for the texture atlas. The atlas layout must know the highest row index used across all animations so it allocates enough rows when slicing the spritesheet into frames.

```rust
impl CharacterEntry {
    pub fn calculate_max_animation_row(&self) -> usize {
        self.animations.values()
            .map(|def| if def.directional { def.start_row + 3 } else { def.start_row })
            .max()
            .unwrap_or(0)
    }
}
```

Directional animations consume 4 rows (start_row through start_row+3), non-directional consume 1.

---

## 6. Animation Engine (`animation.rs`)

### Facing  - Direction Tracking

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum Facing { Up, Left, Down, Right }

impl Facing {
    pub fn from_direction(direction: Vec2) -> Self {
        if direction.x.abs() > direction.y.abs() {
            if direction.x > 0.0 { Facing::Right } else { Facing::Left }
        } else {
            if direction.y > 0.0 { Facing::Up } else { Facing::Down }
        }
    }

    fn direction_index(self) -> usize {
        match self { Up => 0, Left => 1, Down => 2, Right => 3 }
    }
}
```

For diagonal movement, the dominant axis wins. `direction_index()` maps to the spritesheet row offset within a directional animation group.

### AnimationClip  - Frame Range Math

`AnimationClip` maps a spritesheet row to a range of atlas indices. `new()` calculates the first and last frame index from the row and column count, `next()` advances to the next frame and wraps back to the first for looping, and `is_complete()` checks if a one-shot animation (like Jump) has played its last frame.

```rust
pub struct AnimationClip {
    first: usize,   // First frame index in the atlas
    last: usize,    // Last frame index in the atlas
}

impl AnimationClip {
    pub fn new(row: usize, frame_count: usize, atlas_columns: usize) -> Self {
        let first = row * atlas_columns;
        Self { first, last: first + frame_count - 1 }
    }

    pub fn start(self) -> usize { self.first }
    pub fn contains(self, index: usize) -> bool { (self.first..=self.last).contains(&index) }
    pub fn next(self, index: usize) -> usize {
        if index >= self.last { self.first } else { index + 1 }
    }
    pub fn is_complete(self, current_index: usize, timer_finished: bool) -> bool {
        current_index >= self.last && timer_finished
    }
}
```

**Example:** Walk Down at `start_row: 8`, directional, 9 frames, 9 atlas columns.
- Actual row = `8 + Facing::Down.direction_index()` = `8 + 2` = `10`
- `first` = `10 * 9` = `90`
- `last` = `90 + 9 - 1` = `98`
- Frames cycle: 90 → 91 → ... → 98 → 90 (loop)

### Components

These three components work together to drive animation: `AnimationController` tracks which animation clip is active and which direction the character faces, `AnimationState` stores current and previous-frame movement flags for change detection (e.g., detecting the moment a character starts or stops moving), and `AnimationTimer` controls frame playback rate.

```rust
#[derive(Component)]
pub struct AnimationController {
    pub current_animation: AnimationType,  // Walk, Run, or Jump
    pub facing: Facing,                     // Up, Down, Left, Right
}

#[derive(Component, Default)]
pub struct AnimationState {
    pub is_moving: bool,
    pub was_moving: bool,      // Previous frame's value
    pub is_jumping: bool,
    pub was_jumping: bool,
}

#[derive(Component, Deref, DerefMut)]
pub struct AnimationTimer(pub Timer);
```

State change detection: `just_started_moving = is_moving && !was_moving`. The `update_animation_flags` system copies current → previous at frame end.

> **Note:** Boolean flags work for Walk/Run/Jump but don't scale well. Chapter 4 introduces a proper **state machine** pattern.

### The `animate_characters` System

Three branches per frame:

| Condition | Action |
|-----------|--------|
| **Animation changed** (just started/stopped moving or jumping) | Reset to frame 0, update timer duration |
| **Should animate** (currently moving or jumping) | Tick timer, advance frame on completion |
| **Idle** (not moving, not jumping) | Snap to frame 0 (standing pose) |

Safety check: if the current atlas index is outside the clip's range (e.g., after a character switch), reset to clip start.

---

## 7. Movement System (`movement.rs`)

### Input Reading

This function uses an iterator chain to convert key presses into a movement vector: it filters the key-direction pairs to find which keys are currently pressed, extracts their direction vectors, and sums them. The result is a combined direction that naturally handles diagonal movement (e.g., Right + Up = `Vec2(1.0, 1.0)`).

```rust
fn read_movement_input(input: &ButtonInput<KeyCode>) -> Vec2 {
    const MOVEMENT_KEYS: [(KeyCode, Vec2); 4] = [
        (KeyCode::ArrowLeft,  Vec2::NEG_X),
        (KeyCode::ArrowRight, Vec2::X),
        (KeyCode::ArrowUp,    Vec2::Y),
        (KeyCode::ArrowDown,  Vec2::NEG_Y),
    ];
    MOVEMENT_KEYS.iter()
        .filter(|(key, _)| input.pressed(*key))
        .map(|(_, dir)| *dir)
        .sum()
}
```

Diagonal input sums to e.g. `Vec2(1.0, 1.0)`, which gets `.normalize()`d in `move_player` so diagonal movement isn't ~1.41× faster.

### Controls Summary

| Key | Action |
|-----|--------|
| Arrow keys | Move in 4/8 directions |
| Shift (L or R) | Sprint (uses `run_speed_multiplier` from config) |
| Space | Jump (plays once, then returns to Walk) |
| 1–6 | Switch character |

### `move_player` Logic

1. Read direction from arrow keys.
2. If Space just pressed → set `is_jumping`, switch to `AnimationType::Jump`.
3. If direction ≠ zero: normalize, apply speed × delta_time, update Transform and Facing.
4. If not jumping: set `is_moving = true`, animation = Run (if Shift) or Walk.
5. If no input and not jumping: `is_moving = false`, animation = Walk (idle pose).

### `update_jump_state`

Monitors jump animation completion using `AnimationClip::is_complete()`. When the last frame plays and the timer finishes, resets `is_jumping = false` and switches back to `AnimationType::Walk`.

---

## 8. Spawn & Character Switching (`spawn.rs`)

### Two-Stage Spawn Pattern

Bevy loads assets asynchronously. The pattern:

**Stage 1  - `spawn_player` (Startup):**
- Kicks off `asset_server.load("characters/characters.ron")`.
- Stores the `Handle<CharactersList>` in a resource.
- Spawns a bare entity: `Player` marker + `Transform` + empty `Sprite`.

**Stage 2  - `initialize_player_character` (Update, self-terminating):**
- Runs every frame but queries `(With<Player>, Without<AnimationController>)`.
- Once the `.ron` loads: grabs character data, loads texture, creates atlas layout, inserts all components.
- After inserting `AnimationController`, the entity no longer matches the query → system becomes a no-op.

> **Tip:** This self-terminating query pattern is idiomatic Bevy for handling async asset dependencies. Use `Without<SomeMarker>` to gate initialization, then insert that marker when done.

### Character Switching

`switch_character` maps digit keys 1–9 to character indices. On press:
1. Validates index is within `characters_list.characters.len()`.
2. Updates `CurrentCharacterIndex` resource.
3. Replaces `CharacterEntry` component with new character's data (`*current_entry = ...`).
4. Loads new texture, creates new atlas layout, replaces `Sprite`.

The animation and movement systems automatically adapt  - they read from `CharacterEntry`, which just changed.

---

## 9. Plugin Registration (`mod.rs`)

`RonAssetPlugin` registers a loader so Bevy can deserialize `.characters.ron` files into `CharactersList` assets. `init_resource` creates the `CurrentCharacterIndex` with its `Default` value (index 0). The six systems in the tuple run in parallel by default since Bevy detects no conflicting data access between them.

```rust
pub struct CharactersPlugin;

impl Plugin for CharactersPlugin {
    fn build(&self, app: &mut App) {
        app.add_plugins(RonAssetPlugin::<CharactersList>::new(&["characters.ron"]))
            .init_resource::<spawn::CurrentCharacterIndex>()
            .add_systems(Startup, spawn::spawn_player)
            .add_systems(Update, (
                spawn::initialize_player_character,
                spawn::switch_character,
                movement::move_player,
                movement::update_jump_state,
                animation::animate_characters,
                animation::update_animation_flags,
            ));
    }
}
```

In `main.rs`:
```rust
mod characters;
// ...
.add_plugins(characters::CharactersPlugin)
// Remove: .add_plugins(PlayerPlugin)
```

---

## 10. Rust Concepts Introduced

| Concept | What It Means | Example in This Chapter |
|---------|--------------|------------------------|
| `HashMap<K, V>` | Key-value lookup table | `animations: HashMap<AnimationType, AnimationDefinition>` |
| `Serialize` / `Deserialize` | Convert structs to/from text formats (RON, JSON) | `#[derive(Serialize, Deserialize)]` on all config types |
| `Asset` + `TypePath` derives | Register a struct as a loadable Bevy asset | `CharacterEntry` and `CharactersList` |
| Method chaining | Sequential iterator operations | `.values().map(...).max().unwrap_or(0)` |
| `unwrap_or(default)` | Extract `Option` value or use fallback | `max().unwrap_or(0)`  - returns 0 if no animations |
| `_` in pattern matching | Ignore unused tuple/struct fields | `(key, _)` to ignore direction, `(_, dir)` to ignore key |
| Dereference `*` | Access/modify data behind a mutable reference | `*current_entry = character_entry.clone()` |
| `Deref` / `DerefMut` | Auto-dereference a wrapper type | `AnimationTimer(Timer)` auto-delegates to inner `Timer` |
| `Option<Res<T>>` | Resource that might not exist yet | `characters_list_res: Option<Res<CharactersListResource>>` |
| Self-terminating queries | Query with `Without<C>` that stops matching after inserting `C` | `Query<Entity, (With<Player>, Without<AnimationController>)>` |
| Component vs Resource | Component = per-entity data; Resource = global singleton | `AnimationState` (component) vs `CurrentCharacterIndex` (resource) |
| `Default` derive | Auto-generate default values (`0`, `false`, `None`, `""`) | `CurrentCharacterIndex { index: 0 }` via `#[derive(Default)]` |
| `.as_ref()` vs `.as_mut()` | Read-only vs mutable borrow of inner Option value | `sprite.texture_atlas.as_ref()` (read) vs `.as_mut()` (write) |

---

## 11. Tips for Further Improvement

### Immediate Enhancements

1. **Add WASD support**  - extend `MOVEMENT_KEYS` array with `(KeyCode::KeyA, Vec2::NEG_X)`, etc.
2. **Idle animation**  - add an `AnimationType::Idle` variant with a slow breathing/swaying cycle. Switch to it when `!is_moving && !is_jumping` instead of freezing on frame 0.
3. **Attack animations**  - add `AnimationType::Attack` to the enum and config. Use the same one-shot pattern as Jump (play once, return to Walk).
4. **Animation blending**  - instead of hard-cutting between animations, lerp the transition over 2–3 frames for smoother feel.
5. **Speed-scaled animation**  - dynamically adjust `frame_time` based on actual movement speed. Faster movement = faster walk cycle.

### Architecture Improvements

6. **State machine** (covered in Chapter 4)  - replace boolean flags with a proper FSM for cleaner state transitions (Idle → Walk → Run → Jump → Attack → ...).
7. **Character selection UI**  - replace number key switching with a visual character picker (portrait grid, scroll wheel).
8. **Hot-reload config**  - watch `characters.ron` for file changes and reload at runtime. Bevy's asset server supports this with `AssetServer::watch_for_changes()`.
9. **Event-driven transitions**  - emit `AnimationChangedEvent` when switching animations, letting other systems react (play sound effects, spawn particles).
10. **Component separation**  - split `CharacterEntry` into smaller components (`CharacterStats`, `SpriteConfig`, `AnimationMap`) for more granular ECS queries.

### Performance Notes

11. **Atlas layout caching**  - `create_character_atlas_layout` creates a new layout handle on every character switch. Cache layouts per character to avoid redundant allocations.
12. **Query optimization**  - the `animate_characters` system queries all entities with animation components. For large entity counts, consider using change detection (`Changed<AnimationController>`) to skip unchanged entities.
13. **RON parsing**  - happens once at load time and is negligible. For hundreds of characters, consider binary serialization (`bincode`) for faster parsing.

---

## 12. Recommended Crates

### Data & Serialization

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`serde`](https://crates.io/crates/serde) | Serialize/deserialize Rust structs | Foundation for RON, JSON, TOML, bincode, etc. |
| [`bevy_common_assets`](https://crates.io/crates/bevy_common_assets) | Asset loaders for common formats | RON, JSON, TOML, YAML, CSV, MsgPack support |
| [`ron`](https://crates.io/crates/ron) | RON parser/writer | Direct use for custom tooling; `bevy_common_assets` wraps it |
| [`bevy_asset_loader`](https://crates.io/crates/bevy_asset_loader) | Declarative asset loading states | Loading screens, progress tracking, dependency management |

### Animation & State Machines

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_spritesheet_animation`](https://crates.io/crates/bevy_spritesheet_animation) | Declarative sprite animation | Define clips + transitions without manual frame math |
| [`seldom_state`](https://crates.io/crates/seldom_state) | State machine for Bevy entities | Clean replacement for boolean flag state management |
| [`bevy_tweening`](https://crates.io/crates/bevy_tweening) | Tween/easing animations | Smooth transitions between animation states |

### Character Systems

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`leafwing-input-manager`](https://crates.io/crates/leafwing-input-manager) | Input abstraction & remapping | Map actions to keys/gamepad instead of hardcoded `KeyCode` checks |
| [`bevy_rapier2d`](https://crates.io/crates/bevy_rapier2d) / [`avian2d`](https://crates.io/crates/avian2d) | Physics & collision | Next chapter covers collision detection |
| [`bevy_egui`](https://crates.io/crates/bevy_egui) | Immediate-mode UI | Character selection screens, stat displays, debug panels |

### Debug & Development

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy-inspector-egui`](https://crates.io/crates/bevy-inspector-egui) | Runtime entity/component inspector | Inspect `CharacterEntry` and `AnimationState` live |
| [`iyes_perf_ui`](https://crates.io/crates/iyes_perf_ui) | Performance overlay | FPS, entity count, system timing |

---

## 13. Spritesheet Layout Reference

All character spritesheets in this tutorial follow the LPC (Liberated Pixel Cup) convention:

```
Row Layout for Directional Animations (directional: true):
  start_row + 0  →  Up    frames
  start_row + 1  →  Left  frames
  start_row + 2  →  Down  frames
  start_row + 3  →  Right frames

Atlas Index Formula:
  index = (start_row + direction_offset) × atlas_columns + frame_number

Example: Walk Right, frame 3, atlas_columns = 9, start_row = 8
  direction_offset = 3 (Right)
  actual_row = 8 + 3 = 11
  index = 11 × 9 + 3 = 102
```

Non-directional animations (like Jump) use a single row regardless of `Facing`.

---

## API Quick Reference

APIs introduced in this chapter. See the [API Glossary](./api-glossary.md) for all APIs across the book.

| API | Category | Description |
|-----|----------|-------------|
| [`Asset`](https://docs.rs/bevy/0.18.0/bevy/asset/trait.Asset.html) | Assets | Marks a type as a loadable Bevy asset |
| [`TypePath`](https://docs.rs/bevy/0.18.0/bevy/reflect/trait.TypePath.html) | Assets | Runtime type path info (required for `Asset`) |
| [`HashMap<K, V>`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) | Rust std | Key-value container with O(1) lookup |
| [`Hash`](https://doc.rust-lang.org/std/hash/trait.Hash.html) | Rust std | Enables use as HashMap key |
| [`Serialize`](https://docs.rs/serde/latest/serde/trait.Serialize.html) | serde | Serialization to RON, JSON, etc. |
| [`Deserialize`](https://docs.rs/serde/latest/serde/trait.Deserialize.html) | serde | Deserialization from RON, JSON, etc. |

---

## 14. Complete Component Hierarchy

When fully initialized, the player entity has:

| Component | Source | Purpose |
|-----------|--------|---------|
| `Player` | `spawn_player` | Marker for queries (`With<Player>`) |
| `Transform` | `spawn_player` | Position, scale, z-order |
| `Sprite` | `initialize_player_character` | Texture atlas reference + current frame index |
| `CharacterEntry` | `initialize_player_character` | All character data from `.ron` |
| `AnimationController` | `initialize_player_character` | Current animation type + facing direction |
| `AnimationState` | `initialize_player_character` | Movement/jump flags + previous-frame flags |
| `AnimationTimer` | `initialize_player_character` | Frame timing (repeating timer) |