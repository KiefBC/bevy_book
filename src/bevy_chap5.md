# Bevy & Rust Game Development  - Chapter 5 Reference

---

## 1. Chapter Overview

This chapter adds two independent features: a **pickup/inventory system** (walk near items to collect them) and a **smooth follow camera** with a zoomed-in view. It also introduces Rust's **borrow checker** and **ownership model** as core concepts.

### What Gets Built

- `Pickable` component + `Inventory` resource  - distance-based collection with HashMap counting
- `ItemKind` enum  - Plant1–4, TreeStump with display names
- `SpawnableAsset` gains `.with_pickable(ItemKind)` builder method
- Smooth lerp-based camera follow with pixel snapping to prevent grid shimmer
- 2× world scale (32px tiles → 64px rendered) for zoomed-in exploration feel
- Borderless fullscreen window mode
- Centralized config expanded: `pickup`, `camera`, updated `player` and `map` modules

---

## 2. Project Structure After This Chapter

```
src/
├── main.rs                       # Updated: fullscreen, CameraPlugin, InventoryPlugin
├── config.rs                     # Updated: +pickup, +camera modules, scaled values
├── camera/                       # NEW
│   ├── mod.rs                    #   CameraPlugin
│   └── camera.rs                 #   MainCamera marker, setup_camera, follow_camera
├── inventory/                    # NEW
│   ├── mod.rs                    #   InventoryPlugin
│   ├── inventory.rs              #   ItemKind, Pickable, Inventory
│   └── systems.rs                #   handle_pickups
├── state/                        # Unchanged
├── collision/                    # Unchanged
├── characters/                   # Unchanged
└── map/
    ├── assets.rs                 # Updated: +pickable field, create_spawner handles pickables
    ├── rules.rs                  # Updated: .with_pickable() on plants/stumps
    └── generate.rs               # Updated: uses config constants, 2× ASSETS_SCALE, no map_pixel_dimensions
```

**Deleted:** `setup_camera` function from `main.rs` (moved to `camera/camera.rs`), local constants in `generate.rs` (`GRID_X`, `GRID_Y`, `TILE_SIZE`, `map_pixel_dimensions`).

---

## 3. Inventory System (`inventory/`)

### ItemKind Enum

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum ItemKind {
    Plant1,      // "Herb"
    Plant2,      // "Flower"
    Plant3,      // "Mushroom"
    Plant4,      // "Fern"
    TreeStump,   // "Wood"
}
```

`display_name()` returns human-readable strings. `fmt::Display` is implemented so `format!("{}", item)` works directly.

### Pickable Component

```rust
#[derive(Component, Debug)]
pub struct Pickable {
    pub kind: ItemKind,
    pub radius: f32,    // Default: 40.0 from config::pickup::DEFAULT_RADIUS
}
```

Attached to world entities during map generation. Per-item radius allows future items with different pickup ranges (e.g., magnetic orbs with larger radius, treasure chests requiring close proximity).

### Inventory Resource

```rust
#[derive(Resource, Default, Debug)]
pub struct Inventory {
    items: HashMap<ItemKind, u32>,  // Item kind → count
}

impl Inventory {
    pub fn add(&mut self, kind: ItemKind) -> u32 { /* returns new count */ }
    pub fn summary(&self) -> String { /* "Herb: 3, Flower: 1" */ }
}
```

**Why HashMap over Vec?** Counting Plant1 items in a Vec requires scanning the entire list. HashMap stores counts directly  - O(1) lookup and increment via `.entry(kind).or_insert(0)`.

### Pickup Detection (`systems.rs`)

```rust
pub fn handle_pickups(
    mut commands: Commands,
    mut inventory: ResMut<Inventory>,
    player_query: Query<&Transform, With<Player>>,
    pickables: Query<(Entity, &GlobalTransform, &Pickable)>,
) {
    // 1. Get player position
    // 2. Collect items within radius (distance_squared for performance)
    // 3. Despawn collected entities + add to inventory
}
```

**Key details:**
- Uses `distance_squared` instead of `distance` to avoid expensive `sqrt`  - compare `dist² ≤ radius²` instead.
- Uses `GlobalTransform` for pickables (not `Transform`) because items may be children of other entities.
- Collect-then-process pattern: gathers items into a `Vec` before despawning, following Rust's borrow checker best practice of separating reads from writes.

### Integration with Map Generation

`SpawnableAsset` gains a `pickable: Option<ItemKind>` field and `.with_pickable(kind)` builder method.

`create_spawner` updated to handle `(tile_type, pickable)` tuple matching:

| Asset | TileType | Pickable | Result |
|-------|----------|----------|--------|
| `plant_1` | `Grass` | `Plant1` | Walkable + collectible |
| `plant_2` | `Grass` | `Plant2` | Walkable + collectible |
| `plant_3` | `Grass` | `Plant3` | Walkable + collectible |
| `plant_4` | `Grass` | `Plant4` | Walkable + collectible |
| `tree_stump_2` | `Tree` | `TreeStump` | Blocking + collectible |
| `tree_stump_1`, `tree_stump_3` | `Tree` | None | Blocking, not collectible |
| All other tiles | Various | None | Normal collision behavior |

Only `tree_stump_2` is pickable  - others remain pure obstacles. This adds world variety.

---

## 4. Camera System (`camera/`)

### Config Values

```rust
pub mod camera {
    pub const CAMERA_LERP_SPEED: f32 = 6.0;   // Higher = snappier follow
    pub const CAMERA_Z: f32 = 1000.0;          // Above all game layers
}
```

### Follow Camera

```rust
pub fn follow_camera(
    time: Res<Time>,
    player_query: Query<&Transform, (With<Player>, Changed<Transform>)>,
    mut camera_query: Query<&mut Transform, (With<MainCamera>, Without<Player>)>,
) {
    // 1. Early exit if player hasn't moved (Changed<Transform> filter)
    // 2. Early exit if camera within 0.5px of player
    // 3. Lerp toward player: lerp_factor = (SPEED * dt).clamp(0.0, 1.0)
    // 4. Pixel-snap with .round() to prevent grid shimmer
}
```

**Lerp explained:** Each frame, camera moves `lerp_factor` fraction of the remaining distance to the player. At 60fps with speed 6.0: `6.0 × 0.0167 ≈ 0.1`, so camera closes 10% of the gap per frame. This creates smooth exponential decay  - fast when far, slow when close.

**Why `.clamp(0.0, 1.0)`?** At very low framerates (e.g., 2fps), `dt` could be 0.5s → factor = 3.0, which would overshoot. Clamping prevents this.

**Why `.round()` pixel snapping?** Without it, camera lands on fractional coordinates (10.3, 25.7), causing tile edges to render at inconsistent subpixel positions → visible "shimmer" or flickering grid lines.

**`Changed<Transform>` filter:** Bevy skips the entire system when the player hasn't moved. Zero cost when standing still.

---

## 5. World Scale Changes

### Before → After

| Config | Chapter 4 | Chapter 5 | Why |
|--------|-----------|-----------|-----|
| `TILE_SIZE` | 32.0 | 64.0 | Zoomed-in view |
| `PLAYER_SCALE` | 0.8 | 1.2 | Larger player to match scaled world |
| `COLLIDER_RADIUS` | 16.0 | 24.0 | Proportional to new scale |
| `ASSETS_SCALE` | `Vec3::ONE` | `Vec3::new(2.0, 2.0, 1.0)` | 32px sprites rendered at 64px |
| `NODE_SIZE_Z` | 1.0 (local) | 1.0 (from config) | Centralized |
| Window | Fixed size (800×576) | Borderless fullscreen | Immersive camera view |
| Background | `Color::WHITE` | `Color::BLACK` | Cleaner look |

### What Got Deleted from `generate.rs`

- Local `GRID_X`, `GRID_Y`, `TILE_SIZE` constants → now imported from `config::map`
- `map_pixel_dimensions()` function → no longer needed (fullscreen, no window sizing)

---

## 6. Updated `main.rs`

Plugins are registered in dependency order: `StatePlugin` first (provides `GameState` that other systems reference), then `CameraPlugin` and `InventoryPlugin` (independent features), then `CollisionPlugin` (builds the collision map), and finally `CharactersPlugin` (depends on collision and state). `ProcGenSimplePlugin` handles WFC generation and tile spawning.

```rust
fn main() {
    App::new()
        .insert_resource(ClearColor(Color::BLACK))
        .add_plugins(DefaultPlugins
            .set(AssetPlugin { file_path: "src/assets".into(), ..default() })
            .set(WindowPlugin {
                primary_window: Some(Window {
                    title: "Bevy Game".into(),
                    mode: WindowMode::BorderlessFullscreen(MonitorSelection::Current),
                    ..default()
                }),
                ..default()
            })
            .set(ImagePlugin::default_nearest()),
        )
        .add_plugins(ProcGenSimplePlugin::<Cartesian3D, Sprite>::default())
        .add_plugins(state::StatePlugin)
        .add_plugins(CameraPlugin)              // NEW
        .add_plugins(inventory::InventoryPlugin) // NEW
        .add_plugins(collision::CollisionPlugin)
        .add_plugins(characters::CharactersPlugin)
        .add_systems(Startup, setup_generator)   // setup_camera removed (in CameraPlugin)
        .run();
}
```

Plugin order: State → Camera → Inventory → Collision → Characters.

---

## 7. Centralized Config (`config.rs`)  - Full State

Values changed from Chapter 4 because the world is now rendered at 2× zoom: `TILE_SIZE` doubled from 32 to 64 (32px sprites scaled up), `COLLIDER_RADIUS` increased from 16 to 24 to stay proportional, and `PLAYER_SCALE` grew from 0.8 to 1.2 so the character matches the larger tile rendering.

```rust
pub mod player {
    pub const COLLIDER_RADIUS: f32 = 24.0;
    pub const PLAYER_Z_POSITION: f32 = 20.0;
    pub const PLAYER_SCALE: f32 = 1.2;
}

pub mod map {
    pub const TILE_SIZE: f32 = 64.0;
    pub const GRID_X: u32 = 25;
    pub const GRID_Y: u32 = 18;
    pub const NODE_SIZE_Z: f32 = 1.0;
}

pub mod pickup {
    pub const DEFAULT_RADIUS: f32 = 40.0;
}

pub mod camera {
    pub const CAMERA_LERP_SPEED: f32 = 6.0;
    pub const CAMERA_Z: f32 = 1000.0;
}
```

---

## 8. Rust Concepts Introduced

### The Borrow Checker & Ownership

This chapter provides the deepest Rust-specific teaching in the series so far.

**Core rules:**
1. **Many readers OR one writer, never both.** You can have multiple `&T` (immutable borrows) or one `&mut T` (mutable borrow), but not both simultaneously.
2. **References must always be valid.** A reference cannot outlive the data it points to (no dangling pointers).
3. **Borrows end at last use**, not at scope end (Non-Lexical Lifetimes / NLL).

**Why it matters for games:**
- No garbage collector pauses (unlike Python/JS)  - the compiler inserts cleanup code at compile time.
- No dangling pointers (unlike C/C++)  - the compiler rejects code that would access freed memory.
- The collect-then-process pattern (gather items to remove into a Vec, then remove them) exists because you can't mutate a collection while iterating over it.

**In Bevy context:** Bevy's `Commands` is deferred (changes apply after the system finishes), so the strict borrow rule doesn't technically apply to entity despawning. But the collect-then-process pattern is still used as a Rust best practice that works everywhere.

### Other Concepts

| Concept | What It Means | Example |
|---------|--------------|---------|
| `fmt::Display` trait | Implement to enable `{}` formatting for custom types | `impl fmt::Display for ItemKind` |
| `HashMap::entry().or_insert()` | Get-or-create pattern for efficient upsert | `self.items.entry(kind).or_insert(0)` |
| `.truncate()` | Convert `Vec3` → `Vec2` (drop Z) | `transform.translation.truncate()` |
| `distance_squared` | Avoid `sqrt` when only comparing distances | `pos.distance_squared(other) <= radius * radius` |
| `lerp` (linear interpolation) | Smooth transition between two values | `camera_pos.lerp(player_pos, factor)` |
| `.round()` pixel snapping | Prevent subpixel rendering artifacts | `new_pos.x.round()` |
| `Changed<C>` as optimization | Skip system entirely when component unchanged | `Query<..., Changed<Transform>>` |
| `Without<T>` in camera query | Prevent query from matching player entity | `Query<..., (With<MainCamera>, Without<Player>)>` |
| `GlobalTransform` vs `Transform` | World-space position (accounts for parent hierarchy) vs local | `GlobalTransform` for pickables that may be child entities |
| `BorderlessFullscreen` | Fullscreen without window chrome | `WindowMode::BorderlessFullscreen(MonitorSelection::Current)` |
| Tuple matching in `match` | Match on multiple values simultaneously | `match (tile_type, pickable) { (Some(Grass), Some(Plant1)) => ... }` |

---

## 9. Controls Summary (Cumulative)

| Key | Action | State |
|-----|--------|-------|
| Arrow keys | Move | Playing |
| Shift | Sprint | Playing |
| Space | Jump | Playing, grounded |
| 1–6 | Switch character | Playing |
| Escape | Toggle pause | Playing/Paused |
| F3 | Toggle collision debug | Playing (debug only) |
| *Walk near items* | Auto-collect pickups | Playing |

---

## 10. Tips for Further Improvement

### Inventory Enhancements
1. **Inventory UI**  - render collected items as icons with counts (Chapter 6 preview).
2. **Stack limits**  - cap item counts per type, show "Inventory Full" feedback.
3. **Drop items**  - reverse of pickup: spawn entity from inventory back into world.
4. **Crafting**  - combine items (3 Herbs + 1 Mushroom → Potion).
5. **Item rarity/weight**  - extend `ItemKind` or add `ItemDefinition` data file (RON).

### Camera Enhancements
6. **Camera bounds clamping**  - prevent camera from showing areas beyond the map edge. Clamp camera position to `[map_min + half_viewport, map_max - half_viewport]`.
7. **Camera shake**  - on pickup or collision, add temporary random offset for juice.
8. **Zoom controls**  - scroll wheel to adjust `OrthographicProjection.scale`.
9. **Look-ahead**  - offset camera slightly in the player's movement direction for better visibility.
10. **Deadzone**  - only start following when player moves beyond a small radius from camera center.

### Performance Notes
11. **Pickup spatial indexing**  - currently checks distance to every pickable entity every frame. For hundreds of items, use spatial hashing or only check nearby grid cells.
12. **Camera `Changed<Transform>`**  - already optimized; zero cost when player is stationary.
13. **`distance_squared`**  - already optimized; avoids sqrt per-item per-frame.

---

## 11. Recommended Crates

### Inventory & Items
| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_egui`](https://crates.io/crates/bevy_egui) | Immediate-mode UI | Quick inventory panels, item tooltips |
| [`bevy_ui`](https://docs.rs/bevy/latest/bevy/ui/) | Built-in Bevy UI | Node-based layouts for inventory grids |
| [`serde`](https://crates.io/crates/serde) | Serialization | Save/load inventory to disk |

### Camera
| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_pancam`](https://crates.io/crates/bevy_pancam) | Pan/zoom camera | Drag-to-pan, scroll-to-zoom, bounds clamping |
| [`bevy_pixel_camera`](https://crates.io/crates/bevy_pixel_camera) | Pixel-perfect rendering | Eliminates subpixel artifacts without manual `.round()` |
| [`iyes_perf_ui`](https://crates.io/crates/iyes_perf_ui) | Performance overlay | Monitor FPS impact of pickup checks and camera updates |

### World Interaction
| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_mod_picking`](https://crates.io/crates/bevy_mod_picking) | Click/hover detection | Mouse-based item interaction alternative to proximity |
| [`leafwing-input-manager`](https://crates.io/crates/leafwing-input-manager) | Input abstraction | Rebindable "interact" key for pickups instead of auto-collect |

---

## API Quick Reference

APIs introduced in this chapter. See the [API Glossary](./api-glossary.md) for all APIs across the book.

| API | Category | Description |
|-----|----------|-------------|
| [`Entity`](https://docs.rs/bevy/0.18.0/bevy/ecs/entity/struct.Entity.html) | Core ECS | Unique identifier for an ECS entity |
| [`GlobalTransform`](https://docs.rs/bevy/0.18.0/bevy/transform/components/struct.GlobalTransform.html) | Math | World-space transform (parent hierarchy) |
| [`OrthographicProjection`](https://docs.rs/bevy/0.18.0/bevy/camera/struct.OrthographicProjection.html) | Rendering | Camera projection for zoom control |
| [`WindowMode`](https://docs.rs/bevy/0.18.0/bevy/window/enum.WindowMode.html) | Plugins | Fullscreen/windowed/borderless mode |
| [`MonitorSelection`](https://docs.rs/bevy/0.18.0/bevy/window/enum.MonitorSelection.html) | Plugins | Which monitor for fullscreen |
| [`fmt::Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html) | Rust std | User-facing `{}` string formatting |