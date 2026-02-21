# Appendix B: API Glossary

All Bevy, Rust standard library, and third-party APIs referenced across this book, organized by category. Each entry links to official documentation and notes the chapter where it first appears.

---

## Bevy: Core ECS

| API | Description | First Seen |
|-----|-------------|------------|
| [`App`](https://docs.rs/bevy/0.18.0/bevy/app/struct.App.html) | Application builder and entry point | [Ch 1](./bevy_chap1.md) |
| [`Commands`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Commands.html) | Deferred command queue for spawning/despawning entities | [Ch 1](./bevy_chap1.md) |
| [`Component`](https://docs.rs/bevy/0.18.0/bevy/ecs/component/trait.Component.html) | Derive macro to mark a type as attachable to entities | [Ch 1](./bevy_chap1.md) |
| [`Plugin`](https://docs.rs/bevy/0.18.0/bevy/app/trait.Plugin.html) | Trait for modular code organization | [Ch 1](./bevy_chap1.md) |
| [`Query<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Query.html) | System parameter for reading/writing entity components | [Ch 1](./bevy_chap1.md) |
| [`Res<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Res.html) | Immutable access to a global resource | [Ch 1](./bevy_chap1.md) |
| [`ResMut<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.ResMut.html) | Mutable access to a global resource | [Ch 1](./bevy_chap1.md) |
| [`Resource`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/trait.Resource.html) | Derive macro for global singleton data | [Ch 1](./bevy_chap1.md) |
| [`Single<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/system/struct.Single.html) | Query that expects exactly one matching entity | [Ch 1](./bevy_chap1.md) |
| [`Startup`](https://docs.rs/bevy/0.18.0/bevy/app/struct.Startup.html) | Schedule that runs systems once at launch | [Ch 1](./bevy_chap1.md) |
| [`Update`](https://docs.rs/bevy/0.18.0/bevy/app/struct.Update.html) | Schedule that runs systems every frame | [Ch 1](./bevy_chap1.md) |
| [`With<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/query/struct.With.html) | Query filter requiring a component's presence | [Ch 1](./bevy_chap1.md) |
| [`Changed<C>`](https://docs.rs/bevy/0.18.0/bevy/ecs/query/struct.Changed.html) | Filter for modified components | [Ch 4](./bevy_chap4.md) |
| [`Entity`](https://docs.rs/bevy/0.18.0/bevy/ecs/entity/struct.Entity.html) | Unique identifier for an ECS entity | [Ch 5](./bevy_chap5.md) |
| [`OnEnter`](https://docs.rs/bevy/0.18.0/bevy/state/state/struct.OnEnter.html) | One-shot schedule on state entry | [Ch 4](./bevy_chap4.md) |
| [`OnExit`](https://docs.rs/bevy/0.18.0/bevy/state/state/struct.OnExit.html) | One-shot schedule on state exit | [Ch 4](./bevy_chap4.md) |
| [`States`](https://docs.rs/bevy/0.18.0/bevy/state/state/trait.States.html) | Derive for state machine enums | [Ch 4](./bevy_chap4.md) |
| [`Without<T>`](https://docs.rs/bevy/0.18.0/bevy/ecs/query/struct.Without.html) | Query filter excluding entities with a component | [Ch 4](./bevy_chap4.md) |
| [`in_state()`](https://docs.rs/bevy/0.18.0/bevy/state/condition/fn.in_state.html) | Run-condition gating systems to a state | [Ch 4](./bevy_chap4.md) |
| [`.chain()`](https://docs.rs/bevy/0.18.0/bevy/ecs/schedule/trait.IntoSystemConfigs.html) | Enforces sequential system execution | [Ch 4](./bevy_chap4.md) |
| [`.run_if()`](https://docs.rs/bevy/0.18.0/bevy/ecs/schedule/trait.IntoSystemConfigs.html) | Conditional system execution | [Ch 4](./bevy_chap4.md) |

## Bevy: Rendering

| API | Description | First Seen |
|-----|-------------|------------|
| [`Camera2d`](https://docs.rs/bevy/0.18.0/bevy/core_pipeline/core_2d/struct.Camera2d.html) | Marker component for 2D camera entities | [Ch 1](./bevy_chap1.md) |
| [`Color`](https://docs.rs/bevy/0.18.0/bevy/color/enum.Color.html) | Color representation (multiple color spaces) | [Ch 1](./bevy_chap1.md) |
| [`Sprite`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.Sprite.html) | 2D image rendering component | [Ch 1](./bevy_chap1.md) |
| [`Text2d`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.Text2d.html) | 2D text rendering component | [Ch 1](./bevy_chap1.md) |
| [`TextColor`](https://docs.rs/bevy/0.18.0/bevy/text/struct.TextColor.html) | Text color component | [Ch 1](./bevy_chap1.md) |
| [`TextFont`](https://docs.rs/bevy/0.18.0/bevy/text/struct.TextFont.html) | Font configuration for text rendering | [Ch 1](./bevy_chap1.md) |
| [`TextureAtlas`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.TextureAtlas.html) | Sprite atlas frame selection | [Ch 1](./bevy_chap1.md) |
| [`TextureAtlasLayout`](https://docs.rs/bevy/0.18.0/bevy/sprite/struct.TextureAtlasLayout.html) | Defines how a spritesheet is sliced | [Ch 1](./bevy_chap1.md) |
| [`Gizmos`](https://docs.rs/bevy/0.18.0/bevy/prelude/struct.Gizmos.html) | Immediate-mode debug shape drawing | [Ch 4](./bevy_chap4.md) |
| [`OrthographicProjection`](https://docs.rs/bevy/0.18.0/bevy/camera/struct.OrthographicProjection.html) | Camera projection for zoom control | [Ch 5](./bevy_chap5.md) |

## Bevy: Transforms & Math

| API | Description | First Seen |
|-----|-------------|------------|
| [`Transform`](https://docs.rs/bevy/0.18.0/bevy/transform/components/struct.Transform.html) | Position, rotation, and scale of an entity | [Ch 1](./bevy_chap1.md) |
| [`UVec2`](https://docs.rs/bevy/0.18.0/bevy/math/struct.UVec2.html) | Unsigned integer 2D vector | [Ch 1](./bevy_chap1.md) |
| [`Vec2`](https://docs.rs/bevy/0.18.0/bevy/math/struct.Vec2.html) | 2D floating-point vector | [Ch 1](./bevy_chap1.md) |
| [`Vec3`](https://docs.rs/bevy/0.18.0/bevy/math/struct.Vec3.html) | 3D floating-point vector | [Ch 1](./bevy_chap1.md) |
| [`URect`](https://docs.rs/bevy/0.18.0/bevy/math/struct.URect.html) | Unsigned integer rectangle | [Ch 2](./bevy_chap2.md) |
| [`GlobalTransform`](https://docs.rs/bevy/0.18.0/bevy/transform/components/struct.GlobalTransform.html) | World-space transform (parent hierarchy) | [Ch 5](./bevy_chap5.md) |
| [`IVec2`](https://docs.rs/bevy/0.18.0/bevy/math/struct.IVec2.html) | Signed integer 2D vector | [Ch 4](./bevy_chap4.md) |

## Bevy: Assets

| API | Description | First Seen |
|-----|-------------|------------|
| [`AssetServer`](https://docs.rs/bevy/0.18.0/bevy/asset/struct.AssetServer.html) | Loads and manages game assets | [Ch 1](./bevy_chap1.md) |
| [`Assets<T>`](https://docs.rs/bevy/0.18.0/bevy/asset/struct.Assets.html) | Typed storage for loaded assets | [Ch 1](./bevy_chap1.md) |
| [`Handle<T>`](https://docs.rs/bevy/0.18.0/bevy/asset/enum.Handle.html) | Reference-counted pointer to a loaded asset | [Ch 1](./bevy_chap1.md) |
| [`Asset`](https://docs.rs/bevy/0.18.0/bevy/asset/trait.Asset.html) | Marks a type as a loadable Bevy asset | [Ch 3](./bevy_chap3.md) |
| [`TypePath`](https://docs.rs/bevy/0.18.0/bevy/reflect/trait.TypePath.html) | Runtime type path info (required for `Asset`) | [Ch 3](./bevy_chap3.md) |

## Bevy: Input

| API | Description | First Seen |
|-----|-------------|------------|
| [`ButtonInput<KeyCode>`](https://docs.rs/bevy/0.18.0/bevy/input/struct.ButtonInput.html) | Keyboard state resource | [Ch 1](./bevy_chap1.md) |
| [`KeyCode`](https://docs.rs/bevy/0.18.0/bevy/input/keyboard/enum.KeyCode.html) | Identifies a physical keyboard key | [Ch 1](./bevy_chap1.md) |

## Bevy: Time

| API | Description | First Seen |
|-----|-------------|------------|
| [`Time`](https://docs.rs/bevy/0.18.0/bevy/time/struct.Time.html) | Frame timing and delta time resource | [Ch 1](./bevy_chap1.md) |
| [`Timer`](https://docs.rs/bevy/0.18.0/bevy/time/struct.Timer.html) | Countdown or repeating timer | [Ch 1](./bevy_chap1.md) |
| [`TimerMode`](https://docs.rs/bevy/0.18.0/bevy/time/enum.TimerMode.html) | Once or Repeating timer behavior | [Ch 1](./bevy_chap1.md) |

## Bevy: Plugins & Configuration

| API | Description | First Seen |
|-----|-------------|------------|
| [`AssetPlugin`](https://docs.rs/bevy/0.18.0/bevy/asset/struct.AssetPlugin.html) | Configures asset loading paths | [Ch 1](./bevy_chap1.md) |
| [`ClearColor`](https://docs.rs/bevy/0.18.0/bevy/camera/struct.ClearColor.html) | Global background color resource | [Ch 1](./bevy_chap1.md) |
| [`DefaultPlugins`](https://docs.rs/bevy/0.18.0/bevy/struct.DefaultPlugins.html) | Standard plugin group | [Ch 1](./bevy_chap1.md) |
| [`ImagePlugin`](https://docs.rs/bevy/0.18.0/bevy/render/texture/struct.ImagePlugin.html) | Configures image loading defaults | [Ch 1](./bevy_chap1.md) |
| [`Window`](https://docs.rs/bevy/0.18.0/bevy/window/struct.Window.html) | Window configuration | [Ch 1](./bevy_chap1.md) |
| [`WindowPlugin`](https://docs.rs/bevy/0.18.0/bevy/window/struct.WindowPlugin.html) | Configures window creation | [Ch 1](./bevy_chap1.md) |
| [`WindowResolution`](https://docs.rs/bevy/0.18.0/bevy/window/struct.WindowResolution.html) | Window size configuration | [Ch 2](./bevy_chap2.md) |
| [`MonitorSelection`](https://docs.rs/bevy/0.18.0/bevy/window/enum.MonitorSelection.html) | Which monitor for fullscreen | [Ch 5](./bevy_chap5.md) |
| [`WindowMode`](https://docs.rs/bevy/0.18.0/bevy/window/enum.WindowMode.html) | Fullscreen/windowed/borderless mode | [Ch 5](./bevy_chap5.md) |

## Rust Standard Library

| API | Description | First Seen |
|-----|-------------|------------|
| [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html) | Deep-copy capability | [Ch 1](./bevy_chap1.md) |
| [`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html) | Cheap bitwise duplication | [Ch 1](./bevy_chap1.md) |
| [`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html) | Debug formatting (`{:?}`) | [Ch 1](./bevy_chap1.md) |
| [`Default`](https://doc.rust-lang.org/std/default/trait.Default.html) | Default value generation | [Ch 1](./bevy_chap1.md) |
| [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html) | Transparent wrapper delegation | [Ch 1](./bevy_chap1.md) |
| [`DerefMut`](https://doc.rust-lang.org/std/ops/trait.DerefMut.html) | Mutable wrapper delegation | [Ch 1](./bevy_chap1.md) |
| [`Eq`](https://doc.rust-lang.org/std/cmp/trait.Eq.html) | Total equality | [Ch 1](./bevy_chap1.md) |
| [`Option<T>`](https://doc.rust-lang.org/std/option/enum.Option.html) | Optional value (`Some` or `None`) | [Ch 1](./bevy_chap1.md) |
| [`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) | Equality comparison (`==`) | [Ch 1](./bevy_chap1.md) |
| [`Result<T, E>`](https://doc.rust-lang.org/std/result/enum.Result.html) | Success or failure return type | [Ch 1](./bevy_chap1.md) |
| [`Vec<T>`](https://doc.rust-lang.org/std/vec/struct.Vec.html) | Growable heap-allocated array | [Ch 1](./bevy_chap1.md) |
| [`String`](https://doc.rust-lang.org/std/string/struct.String.html) | Owned heap-allocated UTF-8 string | [Ch 2](./bevy_chap2.md) |
| [`HashMap<K, V>`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) | Key-value container with O(1) lookup | [Ch 3](./bevy_chap3.md) |
| [`Hash`](https://doc.rust-lang.org/std/hash/trait.Hash.html) | Enables use as HashMap key | [Ch 3](./bevy_chap3.md) |
| [`fmt::Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html) | User-facing `{}` string formatting | [Ch 5](./bevy_chap5.md) |

## serde

| API | Description | First Seen |
|-----|-------------|------------|
| [`Serialize`](https://docs.rs/serde/latest/serde/trait.Serialize.html) | Serialization to RON, JSON, etc. | [Ch 3](./bevy_chap3.md) |
| [`Deserialize`](https://docs.rs/serde/latest/serde/trait.Deserialize.html) | Deserialization from RON, JSON, etc. | [Ch 3](./bevy_chap3.md) |
