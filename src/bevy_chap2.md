# Bevy & Rust Game Development  - Chapter 2 Reference

---

## 1. Chapter Overview

This chapter builds a **procedurally generated tilemap world** using the **Wave Function Collapse (WFC)** algorithm. By the end you have layered terrain (dirt → grass → yellow grass → water → props) with smooth transitions, all generated at runtime from a set of socket-based compatibility rules.

### What Gets Built

| Layer | Z-Level | Purpose | Weight |
|-------|---------|---------|--------|
| Dirt | 0 | Base foundation  - fills everything | 20.0 |
| Green Grass | 1 | Patches on top of dirt with smooth edges | 5.0 |
| Yellow Grass | 2 | Patches on top of green grass | 5.0 |
| Water | 3 | Lakes/ponds with shoreline transitions | 0.02–0.2 |
| Props | 4 | Trees, rocks, plants (land-only) | 0.008–0.025 |

---

## 2. Wave Function Collapse (WFC)  - How It Works

WFC is a constraint-solving algorithm analogous to Sudoku. The core loop:

1. **Find the most constrained cell**  - the grid position with the fewest valid tile options remaining.
2. **Collapse it**  - pick one tile (weighted random selection).
3. **Propagate constraints**  - update all neighbors to remove now-invalid options.
4. **Repeat** until the grid is full or a contradiction occurs.
5. **On contradiction**  - restart with a new random seed (no backtracking in this implementation).

### Key Properties

- **Deterministic per seed**  - same seed produces the same world every time.
- **Weight-driven**  - higher-weight tiles appear more frequently. Tweaking a single weight value drastically changes the landscape.
- **No large-scale structure**  - WFC is local. It won't create "one big lake" or "a mountain range" by default. You need post-processing or pre-seeding for that.
- **Layer separation simplifies rules**  - instead of one massive ruleset, each terrain type gets its own layer with independent horizontal rules and vertical connections to layers above/below.

### Controlling Randomness

| Technique | Effect |
|-----------|--------|
| Same seed (`RngMode::Seeded(42)`) | Reproducible worlds for testing/sharing |
| Random seed (`RngMode::RandomSeed`) | Unique world each run |
| Tile weights | Bias frequency (e.g., `WATER_WEIGHT = 0.02` → small ponds; `0.07` → large lakes) |
| Grid wrapping | `(true, true, false)` → Pac-Man-style world wrap |

### WFC Limitations to Be Aware Of

- No global structure control without pre-seeding or post-processing.
- Complex rules increase computation time and failure rate.
- Poorly designed socket rules produce broken or unrealistic landscapes.
- Performance degrades with very large grids  - chunking strategies needed (covered in later chapters).

---

## 3. Project Setup

### Dependencies

The `bevy_procedural_tilemaps` crate provides a WFC constraint solver, 3D Cartesian grid types, and a sprite spawner that automatically places tiles as the grid is solved. It handles all the generation plumbing so you only need to define rules and assets.

```toml
[package]
name = "bevy_game"
version = "0.1.0"
edition = "2024"

[dependencies]
bevy = "0.18"
bevy_procedural_tilemaps = "0.2.0"
```

> **Note:** `bevy_procedural_tilemaps` is a fork of `ghx_proc_gen` maintained for Bevy 0.18 compatibility. For 3D support and debug tools, check the original [`ghx_proc_gen`](https://crates.io/crates/ghx_proc_gen).

### File Structure

Data flows through these files as a pipeline: `tilemap.rs` defines sprite coordinates → `assets.rs` loads the atlas and creates renderable sprites → `sockets.rs` defines how tiles connect → `rules.rs` builds WFC models from sockets → `generate.rs` configures the grid and launches the solver.

```
src/
├── main.rs
├── player.rs
└── map/
    ├── mod.rs          # Module declarations
    ├── assets.rs        # SpawnableAsset, atlas loading, sprite conversion
    ├── tilemap.rs       # Sprite atlas coordinates & definitions
    ├── models.rs        # TerrainModelBuilder  - keeps models & assets synced
    ├── sockets.rs       # Socket type definitions for each layer
    ├── rules.rs         # WFC rules per layer + build_world()
    └── generate.rs      # Grid config, generator setup, spawning
```

### Asset Structure

```
src/assets/
├── male_spritesheet.png        # Player (from Ch1)
└── tile_layers/
    └── tilemap.png             # 256×320 sprite atlas, 32×32 tiles
```

> **Tip:** The tilemap assets are based on [16x16 Game Assets by George Bailey](https://opengameart.org/content/16x16-game-assets) (CC-BY 4.0), upscaled to 32×32.
---

## 4. Architecture Overview

The system flows through five stages:

```
tilemap.rs          → Defines sprite names + pixel coordinates in the atlas
    ↓
assets.rs           → Loads atlas, creates TilemapHandles, converts names → Sprites
    ↓
sockets.rs          → Defines socket types per layer (dirt, grass, water, props)
    ↓
rules.rs            → Builds models with socket patterns + connection rules
    ↓
generate.rs         → Configures WFC generator, spawns the map entity
```

### The Socket System

Every tile model exposes **6 sockets** (one per face in a 3D Cartesian grid):

| Direction | Axis | Meaning in 2D |
|-----------|------|----------------|
| `x_pos` | Right | Tile to the right |
| `x_neg` | Left | Tile to the left |
| `y_pos` | Up | Tile above |
| `y_neg` | Down | Tile below |
| `z_pos` | Top layer | What sits on top of this tile (layer above) |
| `z_neg` | Bottom layer | What this tile sits on (layer below) |

Two tiles can be placed adjacent if their touching sockets are declared compatible via `socket_collection.add_connections(...)`.

---

## 5. Step-by-Step Implementation

### Step 1  - Tilemap Definition (`tilemap.rs`)

Defines the sprite atlas layout and provides lookup methods. Each entry maps a human-readable sprite name to its pixel coordinates in the atlas PNG, all resolved at compile time as `const` data with zero runtime overhead.

```rust
pub struct TilemapSprite {
    pub name: &'static str,
    pub pixel_x: u32,
    pub pixel_y: u32,
}

pub struct TilemapDefinition {
    pub tile_width: u32,
    pub tile_height: u32,
    pub atlas_width: u32,
    pub atlas_height: u32,
    pub sprites: &'static [TilemapSprite],
}

pub const TILEMAP: TilemapDefinition = TilemapDefinition {
    tile_width: 32,
    tile_height: 32,
    atlas_width: 256,
    atlas_height: 320,
    sprites: &[
        TilemapSprite { name: "dirt", pixel_x: 128, pixel_y: 0 },
        // ... all grass, water, prop sprites
    ],
};
```

Key methods: `sprite_index(name) → Option<usize>`, `sprite_rect(index) → URect`.

> **Tip:** All sprite data is `const`  - resolved at compile time, zero runtime overhead.

---

### Step 2  - Asset Loading (`assets.rs`)

#### SpawnableAsset

```rust
#[derive(Clone)]
pub struct SpawnableAsset {
    sprite_name: &'static str,
    grid_offset: GridDelta,          // Multi-tile offset (e.g., tree top = (0,1,0))
    offset: Vec3,                     // Fine pixel positioning
    components_spawner: fn(&mut EntityCommands),  // Custom component injection
}
```

Builder pattern: `SpawnableAsset::new("dirt")` or `SpawnableAsset::new("big_tree_tl").with_grid_offset(GridDelta::new(0, 1, 0))`.

#### TilemapHandles

Holds `Handle<Image>` and `Handle<TextureAtlasLayout>` after atlas loading. The `.sprite(index)` method creates a `Sprite` ready for rendering.

#### Loading Pipeline

1. `prepare_tilemap_handles(...)`  - loads the atlas PNG, registers each sprite's rect in the layout.
2. `load_assets(...)`  - converts `Vec<Vec<SpawnableAsset>>` into `ModelsAssets<Sprite>` that the generator consumes.

---

### Step 3  - Socket Definitions (`sockets.rs`)

Each layer gets its own socket struct. The `material` socket enforces same-type adjacency horizontally, while `layer_up`/`layer_down` control vertical stacking between layers. Transition sockets like `void_and_grass`/`grass_and_void` create smooth curved edges where terrain meets empty space.

```rust
pub struct DirtLayerSockets {
    pub layer_up: Socket,
    pub layer_down: Socket,
    pub material: Socket,
}

pub struct GrassLayerSockets {
    pub layer_up: Socket,
    pub layer_down: Socket,
    pub material: Socket,
    pub void_and_grass: Socket,   // Transition: empty → grass
    pub grass_and_void: Socket,   // Transition: grass → empty
    pub grass_fill_up: Socket,    // Allows yellow grass to stack
}

// Yellow grass: 3 sockets (reuses grass edge sockets horizontally)
// Water: 6 sockets (similar pattern to grass)
// Props: 5 sockets (includes big_tree base connections)
```

All sockets are created via `socket_collection.create()` in `create_sockets()`.

**Why transition sockets?** A pair like `void_and_grass` / `grass_and_void` ensures edge tiles only connect where one side is empty and the other is filled. This creates smooth, curved borders instead of hard blocky edges.

---

### Step 4  - Model Builder (`models.rs`)

`TerrainModelBuilder` keeps model indices and asset indices in lockstep  - when the WFC solver collapses a cell to model index N, the renderer looks up asset index N to know which sprite to display. Without this synchronization, tiles would render with the wrong sprites.

```rust
pub struct TerrainModelBuilder {
    pub models: ModelCollection<Cartesian3D>,
    pub assets: Vec<Vec<SpawnableAsset>>,
}
```

`create_model(template, assets)` adds a model and its sprites at the same index  - keeping them synchronized. `into_parts()` splits them for the generator and renderer.

> **Important Rust concept  - Generics & Trait Bounds:**
> ```rust
> pub fn create_model<T>(&mut self, template: T, assets: Vec<SpawnableAsset>)
> where T: Into<ModelTemplate<Cartesian3D>>
> ```
> `T` can be `SocketsCartesian3D::Simple`, `SocketsCartesian3D::Multiple`, or a `ModelTemplate`  - anything that converts into a model template.

---

### Step 5  - Rules (`rules.rs`)

Each layer follows the same pattern:

1. **Create a void model**  - empty space (no sprite) with `void` sockets horizontally.
2. **Create the center tile model**  - `material` sockets on all horizontal sides.
3. **Create edge templates**  - outer corners, inner corners, side edges.
4. **Rotate templates**  - `Rot90`, `Rot180`, `Rot270` around `Direction::ZForward` to get all 4 orientations from 1 definition.
5. **Define connection rules**  - which sockets can connect to which.
6. **Define layer connections**  - `add_rotated_connection()` for vertical links.

#### Template Rotation Pattern

Define one socket layout, rotate it to create 4 variants:

```rust
let corner_out = SocketsCartesian3D::Simple { /* sockets */ }.to_template();

builder.create_model(corner_out.clone(), vec![sprite("corner_out_tl")]);
builder.create_model(corner_out.rotated(Rot90, ZForward), vec![sprite("corner_out_bl")]);
builder.create_model(corner_out.rotated(Rot180, ZForward), vec![sprite("corner_out_br")]);
builder.create_model(corner_out.rotated(Rot270, ZForward), vec![sprite("corner_out_tr")]);
```

> **Key insight:** Rotation shifts the **socket pattern**, not the sprite. Each rotation gets paired with a different sprite that visually matches the rotated sockets.

#### Layer-Specific Notes

| Layer | Special Behavior |
|-------|-----------------|
| **Dirt** | Single model, weight 20.0. `material` ↔ `material` only. |
| **Green Grass** | 3 connection rules create organic patches. Uses `SocketsCartesian3D::Multiple` for `z_pos` to support both `layer_up` and `grass_fill_up`. |
| **Yellow Grass** | Reuses green grass's `material`, `void_and_grass`, `grass_and_void` for horizontal connections. Only defines vertical sockets of its own. |
| **Water** | Very low weight (`0.02`) for small ponds. Void model uses `Multiple` sockets with both `layer_up` and `ground_up` to support props above. |
| **Props** | No rotation needed. Multi-tile trees use `GridDelta` offsets and dedicated base sockets. `props_down` ↔ `water.ground_up` ensures land-only placement. |

#### The `build_world()` Function

Orchestrates everything: it creates all sockets, builds every layer's tile models and their connection rules, then returns the three inputs the WFC generator needs  - spawnable assets, the model collection, and the socket collection.

```rust
pub fn build_world() -> (Vec<Vec<SpawnableAsset>>, ModelCollection<Cartesian3D>, SocketCollection) {
    let mut socket_collection = SocketCollection::new();
    let sockets = create_sockets(&mut socket_collection);
    let mut builder = TerrainModelBuilder::new();

    build_dirt_layer(&mut builder, &sockets, &mut socket_collection);
    build_grass_layer(&mut builder, &sockets, &mut socket_collection);
    build_yellow_grass_layer(&mut builder, &sockets, &mut socket_collection);
    build_water_layer(&mut builder, &sockets, &mut socket_collection);
    build_props_layer(&mut builder, &sockets, &mut socket_collection);

    let (assets, models) = builder.into_parts();
    (assets, models, socket_collection)
}
```

---

### Step 6  - Generator Configuration (`generate.rs`)

The grid is 25×18 tiles at 32px each, producing an 800×576 pixel map that fits a standard window. `GRID_Z` is 5 because there are five terrain layers stacked vertically: dirt, grass, yellow grass, water, and props.

```rust
pub const GRID_X: u32 = 25;
pub const GRID_Y: u32 = 18;
pub const TILE_SIZE: f32 = 32.;
const GRID_Z: u32 = 5;  // dirt + grass + yellow_grass + water + props
```

The `setup_generator` system:

1. Calls `build_world()` to get assets, models, and socket rules.
2. Builds WFC `Rules` with `Direction::ZForward` rotation axis (2D game).
3. Creates a `CartesianGrid` with dimensions `(25, 18, 5)` and no wrapping.
4. Configures the generator: `MinimumRemainingValue` node heuristic + `WeightedProbability` model selection.
5. Loads the sprite atlas and converts to renderable assets.
6. Spawns the generator entity centered on screen with `NodesSpawner`.

The `Transform` offsets by half the map size so the grid is centered on screen (origin at the middle rather than the bottom-left). `NodesSpawner` is the bridge between the WFC solver and rendering  - as each cell is collapsed, it spawns the corresponding sprite entity at the correct position.

```rust
commands.spawn((
    Transform::from_translation(Vec3 {
        x: -TILE_SIZE * grid.size_x() as f32 / 2.,
        y: -TILE_SIZE * grid.size_y() as f32 / 2.,
        z: 0.,
    }),
    grid,
    generator,
    NodesSpawner::new(models_assets, NODE_SIZE, ASSETS_SCALE)
        .with_z_offset_from_y(true),  // Y-based depth sorting
));
```

> **`with_z_offset_from_y(true)`**  - tiles higher on screen render in front, giving natural depth ordering without manual z-sorting.

---

### Step 7  - Main Integration (`main.rs`)

`ProcGenSimplePlugin` registers the WFC solver and tile-spawning systems so the generator entity is processed each frame until the grid is fully collapsed. `ImagePlugin::default_nearest()` disables texture filtering, keeping pixel art tiles crisp instead of blurry.

```rust
mod map;
mod player;

fn main() {
    let map_size = map_pixel_dimensions(); // 800×576 for 25×18 grid

    App::new()
        .insert_resource(ClearColor(Color::WHITE))
        .add_plugins(
            DefaultPlugins
                .set(AssetPlugin { file_path: "src/assets".into(), ..default() })
                .set(WindowPlugin {
                    primary_window: Some(Window {
                        resolution: WindowResolution::new(map_size.x as u32, map_size.y as u32),
                        resizable: false,
                        ..default()
                    }),
                    ..default()
                })
                .set(ImagePlugin::default_nearest()),  // Crisp pixel art
        )
        .add_plugins(ProcGenSimplePlugin::<Cartesian3D, Sprite>::default())
        .add_systems(Startup, (setup_camera, setup_generator))
        .add_plugins(PlayerPlugin)
        .run();
}
```

### Player Z-Order Fix

The player must render above all map layers. The 0.8 scale shrinks the 64px LPC character to roughly match the 32px tile size, and `PLAYER_Z` at 20.0 places the player above all five terrain layers (Z 0–4) so they're never hidden behind tiles.

```rust
// player.rs
const PLAYER_Z: f32 = 20.0;

// In spawn_player:
Transform::from_translation(Vec3::new(0., 0., PLAYER_Z)).with_scale(Vec3::splat(0.8)),
```

---

## 6. Rust Concepts Introduced in This Chapter

| Concept | What It Means | Example |
|---------|--------------|---------|
| `&'static str` | Reference to a string literal that lives for the entire program | Sprite names like `"dirt"` baked into the binary |
| Lifetimes (`'a`) | Rust tracks how long references are valid to prevent use-after-free | `'static` = lives forever, `'a` = lives as long as its scope |
| Closures | Anonymous functions that capture environment variables | `\|_\| {}` (no-op), `\|\| socket_collection.create()` |
| Generics (`<T>`) | Type parameter  - one function works with multiple types | `create_model<T>` accepts any socket format |
| Trait bounds (`where T: Into<...>`) | Constrain generic types to specific capabilities | Ensures `T` can convert into `ModelTemplate` |
| `str` vs `String` | `&str` = borrowed view of text; `String` = owned, heap-allocated | Use `&str` for constants, `String` for dynamic text |
| `fn(...)` vs closures | `fn(...)` is a function pointer (no captures); closures can capture | `components_spawner: fn(&mut EntityCommands)` |
| Implicit returns | Last expression without `;` is the return value | `self` at end of `with_grid_offset()` |
| Encapsulation | Private fields (no `pub`) accessed only through public methods | `GridDelta` fields are private; use `with_grid_offset()` |

---

## 7. Quick Weight Tuning Guide

Weights control tile frequency. Small changes produce dramatic results:

| Weight Value | Effect |
|-------------|--------|
| `WATER_WEIGHT = 0.02` | Small scattered ponds |
| `WATER_WEIGHT = 0.07` | Large lake coverage |
| `WATER_WEIGHT = 0.15` | Mostly water with land islands |
| `dirt.with_weight(20.)` | Dirt dominates base layer (desired) |
| `grass.with_weight(5.)` | Moderate grass coverage |
| `PROPS_WEIGHT = 0.025` | Sparse prop placement |
| `ROCKS_WEIGHT = 0.008` | Rocks rarer than plants |

> **Tip:** Start with very low weights (0.01–0.05) for new terrain types and increase gradually. High weights in complex rulesets can cause WFC contradictions.

---

## 8. Known Issues & Workarounds

| Issue | Workaround |
|-------|-----------|
| **White lines between tiles (Linux/Wayland)** | Run with `WINIT_UNIX_BACKEND=x11 WAYLAND_DISPLAY= cargo run` |
| **Player walks on water** | Collision detection not yet implemented (coming in later chapters) |
| **WFC fails on large grids** | Keep grid ≤ ~50×50 for now; chunking strategies covered later |
| **Sprite blurriness** | Ensure `ImagePlugin::default_nearest()` is set (not linear filtering) |

---

## 9. Recommended Crates for Expanding Procedural Worlds

### Alternative/Complementary Proc-Gen

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`ghx_proc_gen`](https://crates.io/crates/ghx_proc_gen) | Original WFC library | 3D support, debug visualization, more advanced features than the fork |
| [`noise`](https://crates.io/crates/noise) | Perlin/Simplex noise generation | Great for height maps, temperature maps, biome distribution pre-seeding |
| [`bracket-noise`](https://crates.io/crates/bracket-noise) | Game-focused noise library | Part of the bracket-lib ecosystem; cellular automata, Perlin, etc. |
| [`bracket-pathfinding`](https://crates.io/crates/bracket-pathfinding) | A* and Dijkstra pathfinding | Essential for NPC navigation on generated terrain |

### Tilemap Rendering (Alternatives to Raw Sprites)

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_ecs_tilemap`](https://crates.io/crates/bevy_ecs_tilemap) | High-performance ECS-native tilemaps | Renders thousands of tiles in a single draw call; much faster than individual sprite entities for large maps |
| [`bevy_ecs_ldtk`](https://crates.io/crates/bevy_ecs_ldtk) | LDtk level editor integration | Hand-design parts of the world, proc-gen the rest |

### Collision & Physics (Next Steps)

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`avian2d`](https://crates.io/crates/avian2d) | 2D physics for Bevy | Add colliders to water tiles and props to prevent player walkthrough |
| [`bevy_rapier2d`](https://crates.io/crates/bevy_rapier2d) | Rapier physics integration | Battle-tested alternative to Avian |

### World Expansion

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_save`](https://crates.io/crates/bevy_save) | Save/load game state | Persist generated worlds with their seeds |
| [`rand`](https://crates.io/crates/rand) | Random number generation | Seeded RNG for reproducible world generation outside of the WFC library |
| [`bevy_asset_loader`](https://crates.io/crates/bevy_asset_loader) | Declarative asset loading | Loading states with progress bars for large atlas files |

### Visual Polish

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`bevy_hanabi`](https://crates.io/crates/bevy_hanabi) | GPU particle effects | Water splashes, dust when walking, fireflies near trees |
| [`bevy_tweening`](https://crates.io/crates/bevy_tweening) | Tweening/easing animations | Animate water tiles with gentle bob, tree sway |
| [`bevy_pixel_camera`](https://crates.io/crates/bevy_pixel_camera) | Pixel-perfect rendering | Prevents sub-pixel blurriness at any zoom level |

---

## 10. Tips for Further Improvement

### Immediate Enhancements

1. **Pre-seed terrain with noise**  - Use Perlin/Simplex noise to create a height map, then bias WFC weights based on elevation. Low areas get higher water weight, high areas get more rocks.
2. **Add animated water**  - Use a simple system that cycles through 2–3 water sprite variants on a timer, similar to the player animation from Chapter 1.
3. **Camera follow + bounds clamping**  - Make the camera follow the player but clamp to map edges so you never see void outside the world.
4. **Collision layer**  - Tag water and prop entities with a `Collider` component. Add a system that prevents player movement into collider tiles.

### Architecture Improvements

5. **Chunk-based generation**  - Instead of generating the entire map at startup, generate chunks around the player and despawn distant ones. This scales to infinite worlds.
6. **Biome system**  - Define biome presets (forest, desert, swamp) with different weight configurations. Use noise to assign biomes to regions, then generate each region with biome-specific weights.
7. **Separate render and logic layers**  - Use `bevy_ecs_tilemap` for rendering (single draw call) while keeping your WFC logic for generation. This dramatically improves performance for large maps.
8. **Seed display/input**  - Show the current world seed on screen. Let the player input a seed to share worlds.

### Performance Notes

9. **Entity count**  - A 25×18 grid with 5 layers spawns up to 2,250 entities. At 100×100×5 that's 50,000 entities. Use `bevy_ecs_tilemap` or sprite batching for anything beyond ~50×50.
10. **Generation time**  - WFC runs synchronously in `setup_generator`. For large grids, move generation to an async task and show a loading screen.
11. **Debug tools**  - Add `bevy-inspector-egui` to inspect tile entities at runtime. The original `ghx_proc_gen` has built-in debug visualization for WFC state.

---

## 11. Complete Layer Stack Visualization

```
Z=4  Props     [trees, rocks, plants]     ← only on ground_up (no water)
Z=3  Water     [center, edges, corners]   ← low weight, smooth shorelines
Z=2  Yellow    [center, edges, corners]   ← sits on grass_fill_up
Z=1  Grass     [center, edges, corners]   ← patches on dirt
Z=0  Dirt      [single tile]              ← covers everything
```

Each position `(x, y)` has one tile per Z-level. Void models (no sprite) fill empty slots. The WFC solves all layers simultaneously  - a single constraint graph across the full 3D grid.

---

## API Quick Reference

APIs introduced in this chapter. See the [API Glossary](./api-glossary.md) for all APIs across the book.

| API | Category | Description |
|-----|----------|-------------|
| [`URect`](https://docs.rs/bevy/0.18.0/bevy/math/struct.URect.html) | Math | Unsigned integer rectangle |
| [`WindowResolution`](https://docs.rs/bevy/0.18.0/bevy/window/struct.WindowResolution.html) | Plugins | Window size configuration |
| [`String`](https://doc.rust-lang.org/std/string/struct.String.html) | Rust std | Owned heap-allocated UTF-8 string |