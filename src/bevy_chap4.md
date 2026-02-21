# Bevy & Rust Game Development  - Chapter 4 Reference

---

## 1. Chapter Overview

This chapter replaces Chapter 3's polling-based initialization and boolean animation flags with proper **state machines** at two levels: game-wide lifecycle states and per-character behavior states. It then builds a complete **tile-based collision system** with circle colliders, swept collision, debug visualization, and **Y-based depth sorting** for walk-behind occlusion.

### What Gets Built

- **Game states**  - `Loading` → `Playing` ↔ `Paused` with `OnEnter`/`OnExit` schedules, loading screen, pause overlay
- **Character state machine**  - `CharacterState` enum (Idle/Walking/Running/Jumping) replacing boolean flags
- **Architecture refactor**  - `movement.rs` split into `input.rs` (what to do) + `physics.rs` (how to move)
- **Facing component**  - extracted from `AnimationController` into its own ECS component
- **Collision map**  - flat grid built from WFC tiles, circle-vs-rectangle tests, swept collision with wall-sliding
- **Shore detection**  - water tiles adjacent to walkable land become walkable shore
- **Debug overlay**  - F3 toggles: green/red walkability grid, cyan collider circle, yellow grid cell highlight
- **Y-based depth sorting**  - player Z updated dynamically so they render behind trees/objects higher on screen
- **Centralized config**  - `src/config.rs` for all magic numbers (tile size, grid dimensions, collider radius, player scale)

---

## 2. Project Structure After This Chapter

```
src/
├── main.rs
├── config.rs                     # NEW  - centralized constants
├── state/                        # NEW  - game lifecycle
│   ├── mod.rs                    #   StatePlugin, check_assets_loaded, toggle_pause
│   ├── game_state.rs             #   GameState enum (Loading/Playing/Paused)
│   ├── loading.rs                #   Loading screen UI + animation
│   └── pause.rs                  #   Pause overlay UI
├── collision/                    # NEW  - tile collision
│   ├── mod.rs                    #   CollisionPlugin
│   ├── tile_type.rs              #   TileType enum + walkability
│   ├── map.rs                    #   CollisionMap resource (grid, circle tests, sweep)
│   ├── systems.rs                #   build_collision_map, shore conversion
│   └── debug.rs                  #   Debug visualization (debug builds only)
├── characters/
│   ├── mod.rs                    #   CharactersPlugin (updated system chain)
│   ├── config.rs                 #   AnimationType now has #[default] Walk
│   ├── animation.rs              #   REFACTORED  - on_state_change + tick_animations
│   ├── input.rs                  #   NEW  - replaces movement.rs (Player marker, input, state machine)
│   ├── physics.rs                #   NEW  - Velocity component, apply_velocity
│   ├── facing.rs                 #   NEW  - Facing component (extracted from AnimationController)
│   ├── state.rs                  #   NEW  - CharacterState enum
│   ├── collider.rs               #   NEW  - Collider component, validate_movement
│   ├── rendering.rs              #   NEW  - Y-based depth sorting
│   ├── spawn.rs                  #   Updated (new components, config imports)
│   └── config.rs                 #   Updated (AnimationType default)
└── map/
    ├── assets.rs                 #   Updated (TileType per SpawnableAsset)
    ├── rules.rs                  #   Updated (.with_tile_type() on every asset)
    └── (rest unchanged)
```

**Deleted files:** `src/characters/movement.rs` (replaced by `input.rs` + `physics.rs`)

---

## 3. Game States (`state/`)

### The Problem with Polling

Chapter 3's `initialize_player_character` ran every frame in `Update`, checking if assets were loaded. After one useful execution, it became an infinite no-op. This wastes cycles and clutters the system schedule.

### State-Based Schedules

Bevy's `States` trait provides special one-shot schedules:

| Schedule | When It Runs |
|----------|-------------|
| `OnEnter(State)` | Exactly once when entering a state |
| `OnExit(State)` | Exactly once when leaving a state |
| `Update.run_if(in_state(State))` | Every frame while in that state |

The game has three lifecycle phases: `Loading` (wait for assets and show a loading screen), `Playing` (all gameplay systems active), and `Paused` (gameplay frozen with an overlay). The `#[default]` attribute on `Loading` means the game always starts in the loading state.

```rust
#[derive(States, Default, Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum GameState {
    #[default]
    Loading,   // Show loading screen, check assets
    Playing,   // Gameplay active
    Paused,    // Overlay visible, gameplay frozen
}
```

### State Transitions

```
Game Start
    │
    ▼
 Loading ──── OnEnter: spawn_loading_screen
    │           Update: check_assets_loaded + animate_loading
    │           OnExit: despawn_loading_screen + initialize_player_character
    ▼
 Playing ◄──► Paused
   (Esc)       (Esc)
    │            │
    │           OnEnter: spawn_pause_menu
    │           OnExit:  despawn_pause_menu
    │
    └── Update systems run_if(in_state(Playing)):
          All character, collision, and animation systems
```

### Key Implementation Details

**`check_assets_loaded`**  - replaces the self-terminating query pattern. Checks if `CharactersList` asset is loaded; if so, calls `next_state.set(GameState::Playing)`.

**`toggle_pause`**  - runs in both Playing and Paused states via `.run_if(in_state(Playing).or(in_state(Paused)))`. Listens for Escape key.

**Loading screen animation**  - cycles "Loading" → "Loading." → "Loading.." → "Loading..." using `(time.elapsed_secs() * 2.0) as usize % 4`.

**Plugin ordering**  - `StatePlugin` must be added BEFORE `CharactersPlugin` in `main.rs` so the state system is initialized before character systems reference it.

---

## 4. Character State Machine (`characters/state.rs`)

### Why Not Booleans

| Boolean Flags (Ch3) | State Enum (Ch4) |
|---------------------|------------------|
| `is_moving`, `was_moving`, `is_jumping`, `was_jumping` | `CharacterState::Walking`, `::Jumping`, etc. |
| Can have impossible combos (`is_moving && is_jumping`) | Compiler enforces exactly one state |
| Manual change detection (`was_moving = is_moving`) | Bevy's `Changed<CharacterState>` filter |
| N states = 2N booleans + transition math | N states = N enum variants + match |
| Adding a state = 2 new bools + update all if-chains | Adding a state = 1 variant + compiler shows every match to update |

### The Enum

```rust
#[derive(Component, Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum CharacterState {
    #[default]
    Idle,
    Walking,
    Running,
    Jumping,
}

impl CharacterState {
    pub fn is_grounded(&self) -> bool {
        matches!(self, Idle | Walking | Running)
    }
}
```

### State Transition Logic (`input.rs`)

```rust
fn determine_new_state(
    current: CharacterState,
    direction: Vec2,
    is_running: bool,
    wants_jump: bool,
) -> CharacterState {
    match current {
        Jumping => Jumping,                                    // Can't exit until animation completes
        _ if wants_jump && current.is_grounded() => Jumping,   // Jump priority
        _ if direction != Vec2::ZERO => {
            if is_running { Running } else { Walking }
        }
        _ => Idle,
    }
}
```

**Match guards** (`_ if condition =>`) combine pattern matching with boolean logic. Read top-to-bottom as priority: Jumping locks until complete, then jump overrides movement, then movement, then idle.

---

## 5. Architecture Refactor

### What Changed

| Old (Ch3) | New (Ch4) | Why |
|-----------|-----------|-----|
| `movement.rs` (input + physics + animation) | `input.rs` + `physics.rs` | Single responsibility |
| `Facing` inside `AnimationController` | `Facing` as own component | Movement owns direction, animation owns clips |
| `AnimationState` (4 booleans) | `CharacterState` enum | Impossible states unrepresentable |
| `animate_characters` (one big system) | `on_state_change_update_animation` + `animations_playback` | Change detection + per-frame playback separated |
| `update_animation_flags` | Deleted | `Changed<CharacterState>` replaces manual tracking |

### System Chain (Final Order)

The `.chain()` ordering is critical because each step depends on the previous one's output: input determines state and velocity, state drives animation selection, collision adjusts velocity before physics applies it, physics updates the transform, depth sorting reads the new position, and finally animation playback uses the current state and timer.

```rust
.add_systems(Update, (
    input::handle_player_input,              // 1. Read keys → set state + velocity + facing
    spawn::switch_character,                 // 2. Number keys → swap character data
    input::update_jump_state,                // 3. Jump animation done? → Idle
    animation::on_state_change_update_animation, // 4. State changed? → pick animation type
    collider::validate_movement,             // 5. Sweep collision → adjust velocity
    physics::apply_velocity,                 // 6. velocity × dt → Transform
    rendering::update_player_depth,          // 7. Y position → Z depth
    animation::animations_playback,          // 8. Tick timer → advance frame
).chain().run_if(in_state(GameState::Playing)));
```

`.chain()` enforces sequential execution. `.run_if(in_state(Playing))` freezes everything when paused.

### Velocity Component (`physics.rs`)

Separating velocity from input lets the collision system modify velocity before physics applies it  - input says "I want to move here", collision says "you can actually move there", and physics executes the final movement. Each `CharacterState` maps to a clear velocity rule: Idle and Jumping produce zero velocity, Walking uses base speed, and Running multiplies by the character's run speed factor.

```rust
#[derive(Component, Debug, Clone, Copy, Default, Deref, DerefMut)]
pub struct Velocity(pub Vec2);

pub fn calculate_velocity(state: CharacterState, direction: Vec2, character: &CharacterEntry) -> Velocity {
    match state {
        Idle | Jumping => Velocity::ZERO,
        Walking => Velocity(direction.normalize_or_zero() * character.base_move_speed),
        Running => Velocity(direction.normalize_or_zero() * character.base_move_speed * character.run_speed_multiplier),
    }
}

pub fn apply_velocity(time: Res<Time>, mut query: Query<(&Velocity, &mut Transform)>) {
    for (velocity, mut transform) in query.iter_mut() {
        if velocity.is_moving() {
            transform.translation += velocity.0.extend(0.0) * time.delta_secs();
        }
    }
}
```

`apply_velocity` is pure physics  - no game logic, works for any entity with `Velocity + Transform`.

---

## 6. Collision System (`collision/`)

### TileType Enum

`TileType` classifies every tile for collision purposes. `is_walkable()` returns false only for Water, Tree, and Rock  - everything else is passable. `collision_adjustment()` returns a small negative value for trees and rocks, which allows the player to slightly clip corners of these obstacles for a smoother feel.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Default)]
pub enum TileType {
    #[default] Empty,
    Dirt, Grass, YellowGrass, Shore,   // Walkable
    Water, Tree, Rock,                  // Blocked
}

impl TileType {
    pub fn is_walkable(&self) -> bool {
        !matches!(self, Water | Tree | Rock)
    }
    pub fn collision_adjustment(&self) -> f32 {
        match self {
            Tree | Rock => -0.2,  // Negative = allow corner cutting
            _ => 0.0,
        }
    }
}
```

New types default to walkable (safer than defaulting to blocked  - forgetting a type creates passable terrain, not invisible walls).

### CollisionMap Resource

A flat `Vec<TileType>` stored in row-major order with coordinate conversion methods.

**Key methods:**

| Method | Purpose |
|--------|---------|
| `world_to_grid(Vec2) → IVec2` | Screen position → tile coordinates |
| `grid_to_world(i32, i32) → Vec2` | Tile coordinates → screen center of tile |
| `is_walkable(x, y) → bool` | Can entities pass through this tile? |
| `circle_intersects_tile(center, radius, gx, gy) → bool` | Circle-vs-rectangle overlap test |
| `is_circle_clear(center, radius) → bool` | Check all tiles under a circle for obstacles |
| `sweep_circle(start, end, radius) → Vec2` | Swept collision with axis-sliding, returns furthest valid position |

**Origin explained:** The map is centered on screen. A 25×18 grid at 32px tiles is 800×576px, so origin = `(-400, -288)`. Without origin offset, `world_to_grid(-32, 0)` would return tile `(-1, 0)` instead of the correct `(11, 9)`.

### Circle-vs-Rectangle Test

The collision test works by clamping the circle's center to the tile's bounding box to find the closest point on the rectangle, then checking if that point is within the circle's radius. Using `distance_squared` instead of `distance` avoids an expensive square root  - comparing squared values gives the same result.

```rust
let closest = Vec2::new(
    center.x.clamp(tile_min.x, tile_max.x),
    center.y.clamp(tile_min.y, tile_max.y),
);
center.distance_squared(closest) <= radius * radius
```

Uses `distance_squared` to avoid an expensive `sqrt`.

### Swept Collision

Prevents tunneling through thin walls at high speed. Steps along the path in quarter-tile increments:

```
For each step along start → end:
  1. Try full movement → if clear, advance
  2. Try X-only slide → if clear, slide horizontally
  3. Try Y-only slide → if clear, slide vertically
  4. All blocked → stop
```

The axis-sliding (steps 2–3) is what makes wall-sliding feel smooth. Running diagonally into a vertical wall lets you slide along it instead of stopping dead.

### Building the Map (`systems.rs`)

`build_collision_map` runs once (gated by `resource_equals(CollisionMapBuilt(false))`):

1. Waits for WFC tiles to exist (early return if no tiles).
2. Scans all `TileMarker + Transform` entities.
3. Handles multi-layer: keeps only the **topmost** tile per (x,y) position (highest Z wins).
4. Creates `CollisionMap` resource and populates it.
5. Post-processes: shore conversion (water tiles adjacent to walkable land → Shore).

### Shore Conversion

Water tiles touching any walkable neighbor (8-directional check) become `Shore`. Shore is walkable, so players can approach water edges naturally instead of stopping a full tile away.

### Integrating with Map Generation

`SpawnableAsset` gains `.with_tile_type(TileType)` builder method. Every asset in `rules.rs` gets a tile type:

| Asset | TileType | Notes |
|-------|----------|-------|
| Dirt | `Dirt` | Walkable base layer |
| Green/Yellow grass | `Grass`/`YellowGrass` | Walkable |
| Water + all water corners | `Water` | Blocked (edges become Shore) |
| Tree trunks (bottom sprites) | `Tree` | Blocked |
| Tree canopies (top sprites) | None | No collision  - player walks under canopy |
| Rocks | `Rock` | Blocked |
| Plants | `Grass` | Walkable decorations |
| Tree stumps | `Tree` | Blocked |

---

## 7. Player Collider (`characters/collider.rs`)

A circle collider is used instead of a rectangle because circles slide smoothly along walls without catching on tile corners. When the player runs diagonally into a wall, the circle's curved edge naturally deflects along the surface instead of getting stuck on grid seams.

```rust
#[derive(Component, Debug, Clone)]
pub struct Collider {
    pub radius: f32,     // Default: 16.0 (from config::player::COLLIDER_RADIUS)
    pub offset: Vec2,    // Offset from entity center (default: Vec2::ZERO)
}
```

### `validate_movement` System

Runs **after** input sets velocity, **before** physics applies it:

1. Get current collider world position.
2. Calculate desired position: `current + velocity * dt`.
3. Call `collision_map.sweep_circle(current, desired, radius)` → get valid position.
4. If sweep modified the path, adjust velocity so physics applies the corrected movement.

This approach means collision doesn't fight with physics  - it cooperates by adjusting velocity before physics reads it.

---

## 8. Debug Visualization (`collision/debug.rs`)

All debug code is wrapped in `#[cfg(debug_assertions)]`  - completely absent from release builds.

| Key | Action |
|-----|--------|
| F3 | Toggle collision debug overlay |

**What the overlay shows:**
- **Green rectangles** (25% opacity)  - walkable tiles
- **Red rectangles** (40% opacity)  - blocked tiles
- **Cyan circle**  - player's collision radius
- **Yellow rectangle**  - grid cell the player is currently in
- **Yellow line**  - from entity center to collider position (shows offset)
- **Red X**  - appears if player is somehow on an unwalkable tile

Uses Bevy's `Gizmos` API  - shapes auto-disappear each frame, no cleanup needed.

---

## 9. Y-Based Depth Sorting (`characters/rendering.rs`)

### The Problem

Player's Z was fixed at 20.0 (Chapter 1). They always render on top of everything, breaking the illusion when walking behind trees.

### The Solution

Dynamically set player Z based on Y position:
- **Higher Y** (top of screen, "far away") → **lower Z** (drawn behind)
- **Lower Y** (bottom of screen, "close") → **higher Z** (drawn in front)

In the formula below, `player_feet_y` anchors the depth calculation to the character's feet rather than their center (so the head doesn't poke above objects they're standing behind). `t` normalizes the Y position across the map's height to a 0–1 range, and `(1 - t)` inverts it so lower screen positions (closer to the viewer) get higher Z values.

```rust
// Uses feet position, not sprite center, for natural-looking occlusion
let player_feet_y = transform.translation.y - (player_sprite_height / 2.0);
let t = ((player_feet_y - map_y0) / map_height).clamp(0.0, 1.0);
let player_z = PLAYER_BASE_Z + NODE_SIZE_Z * (1.0 - t) + PLAYER_Z_OFFSET;
```

**Why feet position?** Using sprite center would cause the player's head to poke above objects they're visually standing in front of. Feet position matches how we perceive depth in top-down games.

Uses `Changed<Transform>` filter  - only recalculates when the player actually moves.

---

## 10. Centralized Configuration (`config.rs`)

```rust
pub mod player {
    pub const COLLIDER_RADIUS: f32 = 16.0;
    pub const PLAYER_Z_POSITION: f32 = 20.0;
    pub const PLAYER_SCALE: f32 = 0.8;
}

pub mod map {
    pub const TILE_SIZE: f32 = 32.0;
    pub const GRID_X: u32 = 25;
    pub const GRID_Y: u32 = 18;
}
```

Imported throughout the codebase  - single place to tune gameplay feel. `spawn.rs` deletes its local `PLAYER_SCALE` and `PLAYER_Z_POSITION` constants in favor of these.

---

## 11. Complete Player Entity Components

After full initialization in this chapter:

| Component | Added By | Purpose |
|-----------|----------|---------|
| `Player` | `spawn_player` | Marker for input queries |
| `Transform` | `spawn_player` | Position, scale, Z-depth |
| `Sprite` | `initialize_player_character` | Texture atlas + current frame |
| `CharacterEntry` | `initialize_player_character` | Character data from `.ron` |
| `AnimationController` | `initialize_player_character` | Current animation type |
| `CharacterState` | `initialize_player_character` | Idle/Walking/Running/Jumping |
| `Velocity` | `initialize_player_character` | Movement vector (px/sec) |
| `Facing` | `initialize_player_character` | Up/Down/Left/Right |
| `Collider` | `initialize_player_character` | Circle radius + offset |
| `AnimationTimer` | `initialize_player_character` | Frame tick timing |

---

## 12. Rust Concepts Introduced

| Concept | What It Means | Example |
|---------|--------------|---------|
| `States` derive + `init_state` | Register an enum as a Bevy state machine with `OnEnter`/`OnExit` | `#[derive(States)]` on `GameState` |
| `OnEnter(S)` / `OnExit(S)` | One-shot schedules on state transitions | `OnExit(Loading)` runs `initialize_player_character` once |
| `.run_if(in_state(S))` | Gate systems to specific states | Gameplay systems only run in `Playing` |
| `Changed<C>` query filter | Only matches entities whose component `C` was modified this frame | `Changed<CharacterState>` triggers animation update |
| Match guards (`_ if cond =>`) | Combine pattern matching with boolean conditions | `_ if wants_jump && current.is_grounded() => Jumping` |
| `matches!` macro | Check if value matches a pattern, returns bool | `matches!(self, Idle \| Walking \| Running)` |
| `#[cfg(debug_assertions)]` | Compile-time conditional  - code only exists in debug builds | Debug collision overlay |
| `#[inline]` | Hint compiler to inline small hot functions | `xy_to_idx`, `in_bounds` |
| Double dereference `**text` | Unwrap Bevy's change-tracking wrapper + dereference the inner reference | `**text = format!("Loading{}", ...)` |
| Tuple struct (`.0` access) | Newtype wrapper with unnamed field | `DebugCollisionEnabled(pub bool)` → `debug_enabled.0` |
| `HashMap::Entry` API | Efficient insert-or-update without double lookup | `match layer_tracker.entry((x,y)) { Occupied/Vacant }` |
| Builder pattern | Chain `.with_*()` methods to configure a struct | `SpawnableAsset::new("grass").with_tile_type(TileType::Grass)` |
| `Deref`/`DerefMut` on newtype | Auto-delegate to inner type's methods | `Velocity(Vec2)` acts like `Vec2` |
| `normalize_or_zero()` | Normalize vector, return zero if length is ~0 (avoids NaN) | Direction normalization in `calculate_velocity` |
| `resource_equals(R)` | Run-condition that checks a resource's value | `run_if(resource_equals(CollisionMapBuilt(false)))` |

---

## 13. Controls Summary

| Key | Action | State Required |
|-----|--------|---------------|
| Arrow keys | Move 4/8 directions | Playing |
| Shift (L/R) | Sprint | Playing |
| Space | Jump (plays once) | Playing, grounded |
| 1–6 | Switch character | Playing |
| Escape | Toggle pause | Playing or Paused |
| F3 | Toggle collision debug | Playing (debug builds only) |

---

## 14. Known Issues & Tips

### Spawn Issues
- Player may spawn on a blocking tile (tree/rock) due to random WFC generation. Restart to regenerate.
- Reduce `WATER_WEIGHT` to `0.001` in `build_water_layer` to minimize water-spawn risk.

### Tuning Collision Feel

| Constant | Effect |
|----------|--------|
| `COLLIDER_RADIUS` (config.rs) | Larger = more clearance from obstacles, smaller = tighter squeezing |
| `collision_adjustment` (tile_type.rs) | Negative = allow corner cutting. `-0.2` on trees/rocks lets players clip corners |
| `sweep step = tile_size * 0.25` | Smaller = smoother but more expensive. 0.25 is a good balance |

### Performance Notes
- `build_collision_map` uses `HashMap<(i32,i32), (TileType, f32)>` for multi-layer dedup  - runs once.
- `validate_movement` calls `sweep_circle` per moving entity per frame. For many entities, consider spatial hashing.
- Debug gizmos are free in release builds (`#[cfg(debug_assertions)]`).
- `Changed<CharacterState>` and `Changed<Transform>` filters skip unchanged entities automatically.

---

## 15. Recommended Crates

### Collision & Physics
| Crate | Purpose | Notes |
|-------|---------|-------|
| [`avian2d`](https://crates.io/crates/avian2d) | Full 2D physics engine for Bevy | Replaces manual collision with rigid bodies, sensors, joints |
| [`bevy_rapier2d`](https://crates.io/crates/bevy_rapier2d) | Rapier physics integration | Alternative physics engine, more mature ecosystem |
| [`bevy_ecs_tilemap`](https://crates.io/crates/bevy_ecs_tilemap) | High-performance tilemap renderer | Built-in tile queries, better than manual grid for large maps |

### State Machines
| Crate | Purpose | Notes |
|-------|---------|-------|
| [`seldom_state`](https://crates.io/crates/seldom_state) | Entity state machines for Bevy | Transition conditions, state enter/exit hooks per-entity |
| [`bevy_state`](https://docs.rs/bevy/latest/bevy/state/) | Built-in (used in this chapter) | `States`, `OnEnter`, `OnExit`, `run_if(in_state(...))` |

### Debug & Visualization
| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy-inspector-egui`](https://crates.io/crates/bevy-inspector-egui) | Runtime entity/component inspector | Live-edit `Collider`, `CharacterState`, `Velocity` |
| [`bevy_mod_debugdump`](https://crates.io/crates/bevy_mod_debugdump) | Visualize system execution order | Verify `.chain()` ordering is correct |

---

## API Quick Reference

APIs introduced in this chapter. See the [API Glossary](./api-glossary.md) for all APIs across the book.

| API | Category | Description |
|-----|----------|-------------|
| [`States`](https://docs.rs/bevy/0.18.0/bevy/state/state/trait.States.html) | Core ECS | Derive for state machine enums |
| [`OnEnter`](https://docs.rs/bevy/0.18.0/bevy/state/state/struct.OnEnter.html) | Core ECS | One-shot schedule on state entry |
| [`OnExit`](https://docs.rs/bevy/0.18.0/bevy/state/state/struct.OnExit.html) | Core ECS | One-shot schedule on state exit |
| [`in_state()`](https://docs.rs/bevy/0.18.0/bevy/state/condition/fn.in_state.html) | Core ECS | Run-condition gating systems to a state |
| [`.run_if()`](https://docs.rs/bevy/0.18.0/bevy/ecs/schedule/trait.IntoSystemConfigs.html) | Core ECS | Conditional system execution |
| [`.chain()`](https://docs.rs/bevy/0.18.0/bevy/ecs/schedule/trait.IntoSystemConfigs.html) | Core ECS | Enforces sequential system execution |
| [`Changed<C>`](https://docs.rs/bevy/0.18.0/bevy/ecs/query/struct.Changed.html) | Core ECS | Filter for modified components |
| [`Without<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/query/struct.Without.html) | Core ECS | Query filter excluding entities with a component |
| [`Gizmos`](https://docs.rs/bevy/0.18.0/bevy/prelude/struct.Gizmos.html) | Rendering | Immediate-mode debug shape drawing |
| [`IVec2`](https://docs.rs/bevy/0.18.0/bevy/math/struct.IVec2.html) | Math | Signed integer 2D vector |