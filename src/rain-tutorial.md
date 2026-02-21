# Building a Rain System in Bevy  - A Learning Guide

This tutorial walks you through adding a rain weather system to your game, step by step. Each section explains **what** you're doing, **why** it works that way, and gives you pointers to learn more.

Take your time with each step. Run `cargo check` often to catch mistakes early.

---

## Table of Contents

1. [Step 1: Set Up the Weather Module](#step-1-set-up-the-weather-module)
2. [Step 2: Create the Weather State](#step-2-create-the-weather-state)
3. [Step 3: Make It Rain  - Spawning Raindrops](#step-3-make-it-rain--spawning-raindrops)
4. [Step 4: Move and Clean Up Raindrops](#step-4-move-and-clean-up-raindrops)
5. [Step 5: Darken the Screen](#step-5-darken-the-screen)
6. [Step 6: Slow the Player](#step-6-slow-the-player)
7. [Step 7: Spawn Puddles](#step-7-spawn-puddles)
8. [Reference: Z-Ordering Cheat Sheet](#reference-z-ordering-cheat-sheet)
9. [Challenges: Take It Further](#challenges-take-it-further)

---

## Step 1: Set Up the Weather Module

Before writing any rain logic, you need a place for it to live. You'll create a new **module**  - just like you already have `src/map/` and `src/characters/`.

### What to do

1. Create the folder `src/weather/`
2. Create the file `src/weather/mod.rs`
3. Register it in `src/main.rs`

### The module file

In `src/weather/mod.rs`, start with just the plugin skeleton:

```rust
use bevy::prelude::*;

pub struct WeatherPlugin;

impl Plugin for WeatherPlugin {
    fn build(&self, app: &mut App) {
        // We'll add systems here as we build them
    }
}
```

### Register it in main.rs

Add two things to `src/main.rs`:

```rust
mod weather;  // Tell Rust this module exists (next to your other mod declarations)
```

And in the `App::new()` chain:

```rust
.add_plugins(weather::WeatherPlugin)  // Register the plugin
```

### Run `cargo check` now!

You should get a clean compile. If not, fix any issues before moving on.

> **💡 Concept: Plugins**
>
> A `Plugin` in Bevy is just a way to group related setup code. When you call
> `.add_plugins(MyPlugin)`, Bevy calls your `build()` method, which is where you
> register systems, resources, and events.
>
> Look at how `CharactersPlugin` in `src/characters/mod.rs` does this  - your
> `WeatherPlugin` will follow the same pattern.
>
> 📖 Learn more: https://bevyengine.org/learn/quick-start/getting-started/plugins/

---

## Step 2: Create the Weather State

You need a way to track whether it's raining. In Bevy, shared game state lives in **Resources**.

### What to do

1. Create `src/weather/state.rs`
2. Define a `WeatherState` resource
3. Add a system that toggles rain with the `R` key
4. Wire it up in the plugin

### The resource

Think about what you need to track:
- Is it currently raining? (`bool`)
- How much should rain slow the player? (`f32`)

```rust
#[derive(Resource)]
pub struct WeatherState {
    pub is_raining: bool,
    pub rain_speed_multiplier: f32,
}
```

You'll also need a `Default` implementation so Bevy can create it automatically. The 0.7 multiplier means the player moves at 70% speed during rain  - using a multiplier rather than a fixed reduction means it scales naturally with different characters' base speeds.

```rust
impl Default for WeatherState {
    fn default() -> Self {
        Self {
            is_raining: false,
            rain_speed_multiplier: 0.7, // 30% slower when raining
        }
    }
}
```

### The toggle system

Write a system that listens for the `R` key and flips `is_raining`:

```rust
pub fn toggle_rain(
    input: Res<ButtonInput<KeyCode>>,
    mut weather: ResMut<WeatherState>,
) {
    if input.just_pressed(KeyCode::KeyR) {
        weather.is_raining = !weather.is_raining;
        println!("Rain: {}", if weather.is_raining { "ON" } else { "OFF" });
    }
}
```

> **💡 Concept: `Res<T>` vs `ResMut<T>`**
>
> - `Res<T>` gives you **read-only** access to a resource
> - `ResMut<T>` gives you **mutable** (read-write) access
>
> Bevy uses this distinction for its parallel scheduling  - systems that only
> _read_ a resource can run at the same time, but if one _writes_ to it, Bevy
> knows to run it exclusively.
>
> 📖 Learn more: https://bevyengine.org/learn/quick-start/getting-started/resources/

> **💡 Concept: `just_pressed` vs `pressed`**
>
> - `just_pressed()`  - true only on the **first frame** the key goes down
> - `pressed()`  - true **every frame** while the key is held
>
> For a toggle, you want `just_pressed`. For movement (like your arrow keys),
> you want `pressed`. Look at how `src/characters/movement.rs` uses both!

### Wire it up

In `src/weather/mod.rs`:

```rust
pub mod state;

use bevy::prelude::*;
use state::WeatherState;

pub struct WeatherPlugin;

impl Plugin for WeatherPlugin {
    fn build(&self, app: &mut App) {
        app.init_resource::<WeatherState>()
            .add_systems(Update, state::toggle_rain);
    }
}
```

### Run `cargo run` and test it!

Press `R` and you should see "Rain: ON" / "Rain: OFF" in your terminal. Nothing visual yet  - that's next.

> **💡 Tip: `init_resource` vs `insert_resource`**
>
> - `init_resource::<T>()`  - creates the resource using its `Default` impl
> - `insert_resource(my_value)`  - inserts a specific value you provide
>
> Since we implemented `Default` for `WeatherState`, `init_resource` is cleaner.

---

## Step 3: Make It Rain  - Spawning Raindrops

Now for the fun part. Each raindrop will be its own **entity** with a tiny sprite.

### Add a random number dependency

You'll need random numbers for raindrop positions. Add `fastrand` to your `Cargo.toml`:

```toml
[dependencies]
# ... your existing dependencies ...
fastrand = "2"
```

> **💡 Other ways to do this**
>
> - `rand` crate  - the standard Rust RNG library, more features but heavier
> - `bevy_rand`  - Bevy plugin that integrates `rand` as a resource
> - `fastrand`  - tiny, fast, no dependencies. Perfect for our use case.
>
> 📖 Check out the `rand` book if you want to learn about RNG in Rust:
> https://rust-random.github.io/book/

### What to do

1. Create `src/weather/rain.rs`
2. Define `Raindrop` and `RaindropVelocity` components
3. Write a `spawn_raindrops` system

### The components

Each raindrop needs:
- A way to identify it (marker component)
- Its own velocity (so drops move at slightly different speeds)

`Raindrop` is a marker component  - it holds no data, but lets you query for "all raindrop entities." `RaindropVelocity` stores a per-entity velocity vector so each drop falls at a slightly different speed and angle, creating natural visual variation.

```rust
use bevy::prelude::*;
use crate::weather::state::WeatherState;

/// Marker component  - lets us query for "all raindrop entities"
#[derive(Component)]
pub struct Raindrop;

/// Each drop has its own velocity for natural variation
#[derive(Component)]
pub struct RaindropVelocity(pub Vec2);
```

> **💡 Concept: Marker Components**
>
> A component with no data (like `Raindrop` or your existing `Player`) is called
> a "marker component". Its only purpose is to **tag** entities so you can find
> them with queries. You already use this pattern  - look at `TilemapGenerator`
> in `src/map/generate.rs` and `Player` in `src/characters/movement.rs`.

### The spawn system

Think about what this system needs to do each frame:
1. Check if it's raining (if not, do nothing)
2. Count existing raindrops (don't spawn too many)
3. Spawn a few new drops at random positions above the screen

Here's the skeleton  - try filling in the spawn logic yourself:

```rust
pub fn spawn_raindrops(
    mut commands: Commands,
    weather: Res<WeatherState>,
    query: Query<&Raindrop>,  // Used to count existing drops
) {
    if !weather.is_raining {
        return;
    }

    let current_count = query.iter().count();
    let max_drops = 200;
    let drops_per_frame = 4;

    if current_count >= max_drops {
        return;
    }

    // Your screen is 800x576, centered at (0,0)
    // So X ranges from -400 to 400, Y from -288 to 288
    let half_width = 400.0;
    let top_y = 300.0; // slightly above the screen top

    for _ in 0..drops_per_frame {
        // Random X position across the screen width
        let x = fastrand::f32() * half_width * 2.0 - half_width;
        // Start just above the visible area
        let y = top_y + fastrand::f32() * 50.0;

        // Base velocity: slight wind to the right, falling fast
        // Add some randomness so drops don't all move identically
        let vx = 30.0 + (fastrand::f32() * 20.0 - 10.0);
        let vy = -300.0 - fastrand::f32() * 100.0;

        commands.spawn((
            Raindrop,
            RaindropVelocity(Vec2::new(vx, vy)),
            // This is how you make a colored rectangle sprite with no texture
            Sprite {
                color: Color::srgba(0.7, 0.8, 1.0, 0.6),
                custom_size: Some(Vec2::new(2.0, 6.0)),
                ..default()
            },
            Transform::from_xyz(x, y, 50.0),
        ));
    }
}
```

### Register the module and system

In `src/weather/mod.rs`, add:

```rust
pub mod rain;
```

And in the plugin's `build()`, add the system:

```rust
.add_systems(Update, (
    state::toggle_rain,
    rain::spawn_raindrops,
))
```

### Run it and see what happens!

Press `R` and you should see a bunch of tiny blue rectangles appear at the top of the screen... but they won't move yet. That's the next step.

> **💡 Concept: `commands.spawn()`**
>
> `commands.spawn((A, B, C))` creates a new entity with components A, B, and C.
> The components are passed as a **tuple** (that's the double parentheses).
>
> Bevy calls these "bundles"  - any tuple of components works. Notice how your
> existing code in `src/characters/spawn.rs` does the same thing:
> ```rust
> commands.spawn((Player, Transform::from_translation(...), Sprite::default()));
> ```

> **💡 Understanding `Sprite` without a texture**
>
> Normally you load an image for a sprite. But you can also set `custom_size`
> and `color` to create a simple colored rectangle  - no image needed. This is
> perfect for particles, UI elements, or debug visualization.
>
> Try experimenting with different sizes and colors! What does a `(1.0, 10.0)`
> raindrop look like vs `(3.0, 3.0)`?

---

## Step 4: Move and Clean Up Raindrops

Spawning is only half the story. You need to:
1. Move drops downward each frame
2. Remove them when they go off screen

### Move system

This is one of the simplest systems you'll write. For each raindrop, apply its velocity. Each raindrop's position updates by its velocity multiplied by delta time, and the `With<Raindrop>` filter ensures only raindrop entities are affected  - not the player or other sprites.

```rust
pub fn move_raindrops(
    time: Res<Time>,
    mut query: Query<(&mut Transform, &RaindropVelocity), With<Raindrop>>,
) {
    for (mut transform, velocity) in query.iter_mut() {
        transform.translation.x += velocity.0.x * time.delta_secs();
        transform.translation.y += velocity.0.y * time.delta_secs();
    }
}
```

> **💡 Concept: Delta Time**
>
> `time.delta_secs()` gives you the time since the last frame in seconds.
> Multiplying velocity by delta time makes movement **frame-rate independent**  -
> drops fall at the same speed whether you're running at 30fps or 144fps.
>
> Without delta time, faster computers would have faster rain!
> This is the same pattern used in `src/characters/movement.rs` line 63.
>
> 📖 Learn more: search for "frame rate independence game development"

> **💡 Concept: Query Filters with `With<T>`**
>
> `Query<(&mut Transform, &RaindropVelocity), With<Raindrop>>` means:
> - Give me `Transform` and `RaindropVelocity` components...
> - But only from entities that **also** have a `Raindrop` component
>
> The `With<Raindrop>` is a **filter**  - it narrows down which entities match.
> You don't need the `Raindrop` data itself, just its presence. Compare this to
> how `move_player` uses `With<Player>` in `src/characters/movement.rs`.

### Despawn system

Remove drops that fall below the screen, and remove ALL drops when rain stops. The -320.0 threshold is just below the visible screen bottom (-288), giving drops a small buffer so they don't visibly pop out of existence. The `!weather.is_raining` check provides instant cleanup when the player toggles rain off.

```rust
pub fn despawn_raindrops(
    mut commands: Commands,
    weather: Res<WeatherState>,
    query: Query<(Entity, &Transform), With<Raindrop>>,
) {
    for (entity, transform) in query.iter() {
        if transform.translation.y < -320.0 || !weather.is_raining {
            commands.entity(entity).despawn();
        }
    }
}
```

> **💡 Concept: `Entity`**
>
> When you query for `Entity`, you get the entity's **ID**  - a unique handle
> that Bevy uses internally. You need this ID to despawn it:
> `commands.entity(id).despawn()`.
>
> This is the same pattern used in `regenerate_map` in `src/map/generate.rs`.

### Register both systems

In your plugin's `build()`:

```rust
.add_systems(Update, (
    state::toggle_rain,
    rain::spawn_raindrops,
    rain::move_raindrops,
    rain::despawn_raindrops,
))
```

### Run it!

Press `R` and you should now see rain falling. Press `R` again and it should stop and all drops disappear.

**Play with the values!** Try changing:
- `drops_per_frame`  - more = denser rain
- `max_drops`  - higher cap = more on screen
- The velocity values  - make it a storm or a drizzle
- The wind (`vx`)  - try negative values for wind blowing left
- The sprite color and size

---

## Step 5: Darken the Screen

A rain overlay gives atmosphere. This is the simplest part  - one big semi-transparent sprite.

### What to do

1. Create `src/weather/overlay.rs`
2. Spawn/despawn a dark overlay based on rain state

### The system

The overlay spawns when rain starts and despawns when it stops. It sits at Z=45  - above the player (Z=20) but below the raindrops (Z=50)  - so the dimming effect covers the world and player while rain falls visibly on top. The dark blue-black color at 30% opacity creates an atmospheric "overcast" feel.

```rust
use bevy::prelude::*;
use crate::weather::state::WeatherState;

#[derive(Component)]
pub struct RainOverlay;

pub fn manage_rain_overlay(
    mut commands: Commands,
    weather: Res<WeatherState>,
    query: Query<Entity, With<RainOverlay>>,
) {
    let overlay_exists = !query.is_empty();

    if weather.is_raining && !overlay_exists {
        // Spawn a big dark rectangle covering the whole screen
        commands.spawn((
            RainOverlay,
            Sprite {
                color: Color::srgba(0.0, 0.0, 0.1, 0.3), // dark blue-black, 30% opacity
                custom_size: Some(Vec2::new(800.0, 576.0)),
                ..default()
            },
            Transform::from_xyz(0.0, 0.0, 45.0), // above map + player, below rain
        ));
    } else if !weather.is_raining && overlay_exists {
        for entity in query.iter() {
            commands.entity(entity).despawn();
        }
    }
}
```

Register `pub mod overlay;` in `mod.rs` and add `overlay::manage_rain_overlay` to your systems tuple.

> **💡 Tip: Experiment with the overlay**
>
> - Try `Color::srgba(0.0, 0.0, 0.2, 0.4)` for a heavier storm
> - Try `Color::srgba(0.1, 0.1, 0.0, 0.2)` for a yellowish haze
> - What if you placed the overlay at Z=15 (between the map and the player)?
>   The player would appear "above" the darkness. Try it!

> **💡 Concept: Z-ordering in 2D**
>
> In Bevy 2D, the Z value of a `Transform` controls **draw order**. Higher Z
> renders on top. Your game's current layers:
> - Tilemap: Z ~0-5
> - Player: Z=20
>
> By placing the overlay at Z=45, it draws on top of everything except the
> raindrops (Z=50). See the cheat sheet at the end of this guide.

---

## Step 6: Slow the Player

This is a small change to an existing file  - `src/characters/movement.rs`.

### What to do

The `move_player` function currently calculates speed like this (line 62):

```rust
let move_speed = calculate_movement_speed(character, is_running);
```

You need to:
1. Import `WeatherState`
2. Add it as a system parameter
3. Apply the multiplier

### The changes

At the top of `src/characters/movement.rs`, add:

```rust
use crate::weather::state::WeatherState;
```

Add the `weather` parameter to `move_player`:

```rust
pub fn move_player(
    input: Res<ButtonInput<KeyCode>>,
    time: Res<Time>,
    weather: Res<WeatherState>,  // ADD THIS
    mut query: Query</* ... same as before ... */>,
) {
```

Then modify the speed calculation:

```rust
let mut move_speed = calculate_movement_speed(character, is_running);
if weather.is_raining {
    move_speed *= weather.rain_speed_multiplier;
}
```

That's it  - three small changes to one file.

> **💡 Concept: Systems are just functions**
>
> Notice how Bevy systems are regular Rust functions. The parameters tell Bevy
> what data the system needs. When you add `weather: Res<WeatherState>`, Bevy
> automatically provides it. You don't call these functions yourself  - Bevy
> calls them for you with the right arguments.
>
> This is called **dependency injection**. If you've used frameworks in other
> languages, it's the same idea. Bevy's version is compile-time checked though,
> so you get errors at build time if something is wrong.

> **💡 Think about: Why a multiplier?**
>
> We store `rain_speed_multiplier: 0.7` in the resource instead of hardcoding it.
> This means you could later:
> - Make different weather intensities (light rain = 0.9, storm = 0.5)
> - Let the player equip boots that modify the multiplier
> - Gradually change it as rain intensifies
>
> Keeping values as data instead of code is a key pattern in game development.

---

## Step 7: Spawn Puddles

Puddles appear gradually while it's raining and disappear when rain stops.

### What to do

1. Create `src/weather/puddles.rs`
2. Use a **timer** to control spawn rate (not every frame!)
3. Place puddles at random tile positions on the map

### Understanding the map coordinates

Your tilemap is spawned with its bottom-left corner at `(-400, -288)`. Each tile is 32x32 pixels. So the **center** of tile at grid position `(col, row)` is at:

```
world_x = -400.0 + (col as f32 + 0.5) * 32.0
world_y = -288.0 + (row as f32 + 0.5) * 32.0
```

Where `col` ranges from 0 to 24 and `row` from 0 to 17 (your 25x18 grid).

> **💡 Tip: Draw it out**
>
> If coordinate math is confusing, grab paper and sketch it:
> - Origin (0,0) is the screen center
> - The map's bottom-left is at (-400, -288)
> - Each tile is 32px wide and tall
> - Tile (0,0) center is at (-400 + 16, -288 + 16) = (-384, -272)
> - Tile (24,17) center is at (-400 + 24.5*32, -288 + 17.5*32) = (384, 272)

### The timer resource

You don't want to spawn puddles every frame  - that would be 60 puddles per second! Use a `Timer`:

`Puddle` is a marker component for querying puddle entities. `PuddleSpawnTimer` wraps a `Timer` in a `Resource` so it's shared globally rather than per-entity. Note the plugin uses `insert_resource` (with a specific timer value) rather than `init_resource` (which would need a `Default` impl).

```rust
use bevy::prelude::*;
use crate::weather::state::WeatherState;

#[derive(Component)]
pub struct Puddle;

#[derive(Resource)]
pub struct PuddleSpawnTimer(pub Timer);
```

Register it in your plugin:

```rust
.insert_resource(puddles::PuddleSpawnTimer(
    Timer::from_seconds(0.8, TimerMode::Repeating)
))
```

> **💡 Concept: Timers**
>
> `Timer::from_seconds(0.8, TimerMode::Repeating)` creates a timer that
> fires every 0.8 seconds. You need to **tick** it each frame with
> `timer.tick(time.delta())`, then check `timer.just_finished()`.
>
> `TimerMode::Repeating` means it auto-resets. `TimerMode::Once` would fire
> once and stop. Look at how `AnimationTimer` works in your
> `src/characters/animation.rs`  - same concept!

### The spawn system

Try writing this one yourself! Here's what it needs to do:
1. Check if it's raining
2. Tick the timer, return if it hasn't fired yet
3. Count existing puddles, return if at the cap (e.g., 20)
4. Pick a random tile position
5. Calculate the world coordinates
6. Spawn a semi-transparent blue sprite there

```rust
pub fn spawn_puddles(
    mut commands: Commands,
    weather: Res<WeatherState>,
    time: Res<Time>,
    mut timer: ResMut<PuddleSpawnTimer>,
    query: Query<&Puddle>,
) {
    if !weather.is_raining {
        return;
    }

    timer.0.tick(time.delta());
    if !timer.0.just_finished() {
        return;
    }

    if query.iter().count() >= 20 {
        return;
    }

    // TODO: Pick a random tile position and spawn a puddle sprite
    // Hint: Use fastrand::u32(0..25) for column, fastrand::u32(0..18) for row
    // Hint: Puddle sprite should be slightly smaller than a tile (28x28)
    // Hint: Use Z=12 so it appears above the map but below the player
    // Hint: Use a semi-transparent blue color
}
```

### The despawn system

When rain stops, all puddles are removed instantly. This is the simplest approach  - for a more polished effect, see the "Puddle fade-out" challenge below where puddles gradually become transparent before being despawned.

```rust
pub fn despawn_puddles(
    mut commands: Commands,
    weather: Res<WeatherState>,
    query: Query<Entity, With<Puddle>>,
) {
    if weather.is_raining {
        return;
    }

    for entity in query.iter() {
        commands.entity(entity).despawn();
    }
}
```

Register `pub mod puddles;` in `mod.rs` and add both systems.

> **💡 Think about: Why not modify the actual tilemap?**
>
> You _could_ change actual tile sprites in the WFC-generated map to show water.
> But the WFC tilemap is complex and procedurally generated  - mutating it would
> mean tracking which tiles you changed and reverting them later.
>
> Instead, puddles are **separate entities** that float above the map visually.
> This is a common game dev pattern: **layer visual effects on top** rather than
> modifying the underlying data. It's simpler and avoids bugs.

---

## Reference: Z-Ordering Cheat Sheet

Here's how your game's visual layers stack up (higher Z = drawn on top):

```
Z=50  Raindrops        (always visible, above everything)
Z=45  Rain overlay      (darkens everything below it)
Z=20  Player character
Z=12  Puddles           (above map, below player)
Z=0-5 Tilemap layers    (dirt → grass → yellow_grass → water → props)
```

> **💡 Tip**
>
> If something isn't visible, it's probably behind something else. Add
> `println!("Z: {}", transform.translation.z)` to debug. Or try setting
> Z=100 temporarily to see if the entity exists but is hidden.

---

## Challenges: Take It Further

Once you have rain working, here are ideas to keep learning:

### Easy
- **Adjust rain intensity**: Make `drops_per_frame` and `max_drops` fields on `WeatherState` so you can change density at runtime
- **Wind direction**: Add a `wind` field to `WeatherState` and use it in the velocity calculation. Try making wind gradually change over time
- **Print puddle count**: Add a system that prints the current puddle count when it changes  - practice with Bevy's `Changed<T>` query filter

### Medium
- **Fade overlay in/out**: Instead of instantly appearing, gradually change the overlay's alpha from 0.0 to 0.3 over 1 second. You'll need to query the overlay's `Sprite` and modify its `color` each frame
- **Puddle fade-out**: When rain stops, instead of instantly despawning puddles, fade their alpha to 0 and then despawn. Add a `FadingOut` component to track this
- **Sound effect**: Add a looping rain sound using Bevy's `AudioPlayer` component. Check out: https://bevyengine.org/learn/quick-start/getting-started/audio/

### Hard
- **Splash particles**: When a raindrop hits the ground (Y reaches the map surface), spawn a tiny "splash" entity that expands and fades out
- **Puddles only on grass**: Tag grass tiles with a marker component during WFC generation (using the `components_spawner` callback in your `Spawnable` struct in `src/map/assets.rs`) and query those positions for puddle placement
- **Day/night + weather cycle**: Create a time-of-day system that automatically triggers rain periodically, changes lighting, etc.

### Rust Learning Opportunities
- **Enums for weather types**: Replace `is_raining: bool` with `enum Weather { Clear, Rain, Storm }`. Learn about Rust enums and pattern matching: https://doc.rust-lang.org/book/ch06-00-enums.html
- **Configuration file**: Move weather constants (speed multiplier, max drops, colors) into a RON file like you did with characters. Practice with `serde` deserialization
- **System sets and ordering**: Learn about Bevy's system ordering to guarantee `toggle_rain` runs before other weather systems: https://bevyengine.org/learn/quick-start/getting-started/system-order-of-execution/

---

## Quick Checklist

Use this to track your progress:

- [ ] Created `src/weather/mod.rs` with `WeatherPlugin`
- [ ] Registered module and plugin in `src/main.rs`
- [ ] `cargo check` passes
- [ ] Created `src/weather/state.rs` with `WeatherState` + `toggle_rain`
- [ ] Press R toggles rain (check terminal output)
- [ ] Added `fastrand = "2"` to `Cargo.toml`
- [ ] Created `src/weather/rain.rs` with raindrop spawning
- [ ] Raindrops appear when pressing R
- [ ] Added `move_raindrops` and `despawn_raindrops`
- [ ] Rain falls and cleans up properly
- [ ] Created `src/weather/overlay.rs`  - screen darkens when raining
- [ ] Modified `src/characters/movement.rs`  - player slows in rain
- [ ] Created `src/weather/puddles.rs`  - puddles appear gradually
- [ ] Full cycle works: R on → rain + dark + puddles + slow → R off → all gone

---

> **Note:** All Bevy and Rust APIs used in this tutorial were introduced in earlier chapters. See the [API Glossary](./api-glossary.md) for a complete reference with documentation links.
