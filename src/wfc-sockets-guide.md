# WFC Sockets Explained  - A Mental Model

This guide explains how the socket system in `src/map/rules.rs` and `src/map/sockets.rs` works. It uses a plug-and-outlet analogy to make the constraint rules intuitive.

---

## The Core Idea

The WFC (Wave Function Collapse) solver must fill **every cell** in the grid with exactly one tile model. Two tiles can be placed next to each other only if their touching faces have compatible sockets.

Think of sockets as **plugs and outlets**. Each face of a tile has a plug. Each connection rule declares which plugs fit which outlets. If the plug doesn't fit, those two tiles can't be neighbors.

---

## The Grid is 3D

Even though the game looks 2D, the grid has three axes:

| Direction | Axis | Meaning |
|-----------|------|---------|
| `x_pos` / `x_neg` | Horizontal | Tile to the right / left |
| `y_pos` / `y_neg` | Vertical | Tile above / below (on screen) |
| `z_pos` / `z_neg` | Layers | What sits on top / what this sits on |

The five Z layers stack like a cake:

```
Z=4  Props          [trees, rocks, plants]
Z=3  Water          [ponds with shorelines]
Z=2  Yellow Grass   [patches on green grass]
Z=1  Green Grass    [patches on dirt]
Z=0  Dirt           [solid foundation, always filled]
```

---

## Socket Types by Role

### 1. `void`  - "I am empty space"

A single socket shared across all layers above dirt. A void tile renders nothing  - no sprite. It's the "air" tile that fills cells where no terrain exists.

**Connection rule:** `void ↔ void`. Air can only neighbor air horizontally. This prevents terrain from appearing without proper edges.

**Why it exists:** The solver must fill every cell. On the grass layer (Z=1), maybe 80 out of 450 cells have grass. The other 370 still need a tile  - that tile is void.

```
A column with grass:          A column without grass:

Z=1  [grass_center]  visible   Z=1  [void]   invisible
Z=0  [dirt]          visible   Z=0  [dirt]    visible (bare dirt)
```

Dirt doesn't have a void model because every Z=0 cell is always dirt  - there's no other option.

### 2. `material`  - "Solid terrain, same type all around"

Each terrain type has its own `material` socket. A tile with `material` on all four horizontal faces is a **center tile**  - fully surrounded by the same terrain.

**Connection rule:** `material ↔ material` (per terrain type). Grass center connects to grass center. Water center connects to water center. They never mix  - grass `material` and water `material` are different plugs.

### 3. Transition Pairs  - "Smooth edges"

These come in pairs and force the solver to place proper edge tiles between terrain and void:

- `void_and_grass`  - "void on one side, grass on the other"
- `grass_and_void`  - "grass on one side, void on the other"

**Connection rule:** `void_and_grass ↔ grass_and_void`. These two plugs only fit each other.

This prevents hard blocky edges. Without transitions, you'd get:

```
BAD:   [void] [grass_center]     ← grass starts abruptly, no border

GOOD:  [void] [grass_edge] [grass_center]   ← smooth transition
```

The edge tile has `void` on one face and `void_and_grass` on the other. The void tile's plug fits the `void` face. The grass center's `material` won't fit `void`, so the solver is forced to insert an edge.

### 4. `layer_up` / `layer_down`  - "Vertical stacking"

These control which layers can sit on top of each other. The connections form a chain:

```
dirt.layer_up       ↔  grass.layer_down
grass.layer_up      ↔  yellow_grass.layer_down
yellow_grass.layer_up ↔  water.layer_down
water.layer_up      ↔  props.layer_down
```

Each layer's void model and terrain models all share the same `layer_down` socket for their layer, so dirt doesn't care whether grass or void sits above it  - both fit.

---

## The Three Edge Shapes

Each terrain type (grass, yellow grass, water) defines three edge templates, then rotates each one four times to cover all orientations.

### Outer Corner

The tip of a terrain peninsula. Two faces are void, two are transitions.

```
        void
         │
void ── [TL] ── void_and_grass
         │
    grass_and_void
```

From the code (`green_grass_corner_out`):
```rust
x_pos: terrain_sockets.grass.void_and_grass,   // right: transition to grass
x_neg: terrain_sockets.void,                    // left: empty
y_pos: terrain_sockets.void,                    // up: empty
y_neg: terrain_sockets.grass.grass_and_void,    // down: transition from grass
```

Rotated four times to create TL, BL, BR, TR corners  - each paired with the matching sprite.

### Inner Corner

The inside bend of an L-shaped terrain area. Two faces are solid material, two are transitions.

```
    grass_and_void
         │
material ── [tile] ── material
         │
    void_and_grass
```

### Side Edge

A straight border. One face is material, the opposite is void, and the two remaining faces are the transition pair.

```
        void
         │
grass_and_void ── [tile] ── void_and_grass
         │
      material
```

### Why Rotation Works

You define the socket pattern once for one orientation. Rotating 90° shifts which face has which socket. Each rotation gets paired with a different sprite (`_tl`, `_bl`, `_br`, `_tr` or `_t`, `_l`, `_b`, `_r`). The rotation moves the plugs around; the sprite visually matches.

---

## How Props Stay Off Water

This is the trickiest pattern. The water layer's void model has **two** plugs on its top face:

```rust
// Water VOID (no water here  - land)
z_pos: vec![water.layer_up, water.ground_up]   // two outlets on top
```

The water center tile has only **one**:

```rust
// Water CENTER (actual water)
z_pos: water.layer_up                          // one outlet on top
```

Props have two different bottom plugs:
- Props void uses `layer_down` → fits `layer_up` (generic stacking)
- Trees/rocks/plants use `props_down` → fits `ground_up` (land-only)

```
Land column:                    Water column:

Z=4  [tree]                     Z=4  [props void]
     plug: props_down                plug: layer_down
          ↕                               ↕
     outlet: ground_up  ✓           outlet: layer_up  ✓
Z=3  [water void]               Z=3  [water center]
     has ground_up                   NO ground_up

Tree fits on land ✓              Tree can't fit on water ✗
                                 (no ground_up outlet)
```

The water center tile simply doesn't offer the `ground_up` outlet. A tree's `props_down` plug has nowhere to connect. The only tile whose bottom plug fits `layer_up` alone is the props void model  - so above water, you always get empty space.

---

## How Big Trees Stay Together

Big trees span two grid cells horizontally (left half + right half). A special socket glues them:

```
Left half:                    Right half:

x_pos: big_tree_1_base        x_neg: big_tree_1_base
(right face)                  (left face)
         └────── must match ──────┘
```

The connection rule says `big_tree_1_base ↔ big_tree_1_base`. No other tile has this plug, so if the solver places a left half, the only thing that fits to its right is the matching right half.

All other faces on both halves are `void`  - the tree sits isolated with air around it.

### Variant Mixing

The `Vec` in connection rules controls whether variants can mix. If you had a mossy trunk variant:

**Mixing allowed** (moss can appear on either half independently):
```rust
(big_tree_1_base, vec![big_tree_1_base, big_tree_1_mossy_base]),
(big_tree_1_mossy_base, vec![big_tree_1_base, big_tree_1_mossy_base]),
```

**No mixing** (mossy left must pair with mossy right):
```rust
(big_tree_1_base, vec![big_tree_1_base]),
(big_tree_1_mossy_base, vec![big_tree_1_mossy_base]),
```

The `Vec` answers: "does it look right if these two are next to each other?"

---

## Yellow Grass: Reusing Another Layer's Plugs

Yellow grass reuses green grass's horizontal sockets (`material`, `void_and_grass`, `grass_and_void`) instead of defining its own. This means yellow grass edges automatically follow the same transition rules as green grass.

The only unique socket is `yellow_grass_fill_down`, which connects to `grass.grass_fill_up`. This ensures yellow grass only appears on top of **solid** green grass (center tiles), never on grass edges or void.

---

## Connection Rules Summary

| Rule | What it means |
|------|---------------|
| `void ↔ void` | Empty space neighbors empty space |
| `material ↔ material` | Solid terrain neighbors same terrain |
| `void_and_grass ↔ grass_and_void` | Transitions pair up for smooth edges |
| `dirt.layer_up ↔ grass.layer_down` | Grass layer sits on dirt layer |
| `grass.grass_fill_up ↔ yellow_grass.yellow_grass_fill_down` | Yellow grass only on solid green grass |
| `water.ground_up ↔ props.props_down` | Props only on land (not water) |
| `big_tree_1_base ↔ big_tree_1_base` | Tree halves must be side by side |

---

## Quick Reference: Reading a Socket Block

When you see a block like this:

```rust
SocketsCartesian3D::Simple {
    x_pos: terrain_sockets.grass.void_and_grass,
    x_neg: terrain_sockets.void,
    z_pos: terrain_sockets.grass.layer_up,
    z_neg: terrain_sockets.grass.layer_down,
    y_pos: terrain_sockets.void,
    y_neg: terrain_sockets.grass.grass_and_void,
}
```

Read it as: "This tile has these plugs on each face. The solver will only place tiles next to it whose plugs are declared compatible in `add_connections`."

- Horizontal faces (`x_pos`, `x_neg`, `y_pos`, `y_neg`) → what can be next to this tile on the same layer
- Vertical faces (`z_pos`, `z_neg`) → what layer sits above/below this tile
- `void` on a face → only other void tiles can be there
- `material` on a face → only same-terrain center tiles can be there
- Transition socket on a face → only the matching transition partner can be there