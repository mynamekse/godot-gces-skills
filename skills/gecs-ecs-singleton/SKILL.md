---
name: gecs-ecs-singleton
description: ECS singleton — the global entry point for GECS. Covers ECS.world, process(), serialize/deserialize, debug mode, and entity preprocessors/postprocessors.
---

# GECS — ECS Singleton

## Overview

`ECS` is an **autoloaded singleton** (`ecs.gd`) that acts as the global façade for the entire GECS framework. All game code accesses the active world, triggers processing, and toggles debug mode through it.

## Source: `addons/gecs/ecs/ecs.gd`

```
class_name _ECS
extends Node
```

## Key Properties

| Property | Type | Description |
|---|---|---|
| `ECS.world` | `World` | The currently active World node. Set this to register a world. |
| `ECS.debug` | `bool` | Enables debug overlay, assertions, and editor debugger messages. |
| `ECS.entity_preprocessors` | `Array[Callable]` | Callables run on each entity **before** it is processed. |
| `ECS.entity_postprocessors` | `Array[Callable]` | Callables run on each entity **after** it is processed. |

## Key Methods

### `ECS.process(delta, group?)`
Delegates to `ECS.world.process(delta, group)`. Call this every frame (or per-group) to tick all systems in the world.

```gdscript
func _process(delta):
    ECS.process(delta)

# Or process only a named group:
func _physics_process(delta):
    ECS.process(delta, "physics")
```

### `ECS.serialize()`
Returns a `Dictionary` snapshot of the entire world (all entities + components). Uses `world.serialize()` internally.

### `ECS.save(path)`
Calls `serialize()` then writes the JSON to `path` via FileAccess.

### `ECS.deserialize(data)`
Restores world state from a snapshot dictionary. Calls `world.deserialize(data)`.

### `ECS.load_save(path)`
Reads JSON from `path` and calls `deserialize()`.

## Setting Up the World

```gdscript
# In your main scene's _ready():
func _ready():
    ECS.world = $World   # World node in the scene tree
    ECS.debug = OS.has_feature("editor")
```

## Debug Mode

When `ECS.debug = true`:
- Assertions fire for bad API usage (e.g. wrong component types).
- Editor Debugger messages (entity_component_added, etc.) are sent.
- Performance markers (`perf_mark`) are recorded per query.
- Archetype explosion warning fires when >500 archetypes exist.

## Preprocessors / Postprocessors

```gdscript
# Add a global pre-process hook (e.g. for networking)
ECS.entity_preprocessors.append(func(entity: Entity):
    if entity.has_component(C_NetDirty):
        sync_entity(entity)
)
```

## Rules

- Always set `ECS.world` before calling `ECS.process()`.
- `ECS.world` must be a `World` node in the scene tree.
- Call `ECS.process(delta)` from `_process` or `_physics_process`, not both (unless using groups).

## See Also

- [`gecs-archetype-world`](../gecs-archetype-world/SKILL.md) — World internals
- [`gecs-system`](../gecs-system/SKILL.md) — System groups and processing
- [`gecs-serialization`](../gecs-serialization/SKILL.md) — Save/load

