---
name: gecs-entity
description: Entity — the core game object in GECS. Covers lifecycle (ready, destroy, enable/disable), adding/removing components, relationships, batch operations, and ecs_id.
---

# GECS — Entity

## Overview

`Entity` (`entity.gd`) extends `Node`. It is the **game object** in GECS — a container for `Component` resources and `Relationship` data. Entities are registered with a `World` via `world.add_entity()`.

## Source: `addons/gecs/ecs/entity.gd`

```
class_name Entity
extends Node
```

## Key Properties

| Property | Type | Description |
|---|---|---|
| `id` | `String` | UUID string (stable, set on `_ready`). |
| `ecs_id` | `int` | Stable integer ID assigned by World when registered (0 until registered). |
| `components` | `Dictionary` | Maps `int` (Script instance_id) → `Component`. Internal. |
| `relationships` | `Array[Relationship]` | All current relationships on this entity. |
| `enabled` | `bool` | Whether this entity is active. Affects query filtering. |

## Lifecycle Overrides

| Method | When Called |
|---|---|
| `define_components()` | Override to return initial `Array[Component]` for this entity type. |
| `on_ready()` | Called at end of `_ready()`, after components are initialized. |
| `on_destroy()` | Called when `world.remove_entity(entity)` is about to free it. |
| `on_enable()` | Called when `entity.enabled` flips to `true`. |
| `on_disable()` | Called when `entity.enabled` flips to `false`. |

```gdscript
class_name E_Player
extends Entity

func define_components() -> Array:
    return [C_Health.new(100), C_Speed.new(5.0)]

func on_ready() -> void:
    print("Player entity ready, ecs_id=", ecs_id)

func on_destroy() -> void:
    print("Player entity destroyed")
```

## Adding / Removing Components

```gdscript
# Single add (triggers archetype move + observer dispatch)
entity.add_component(C_Poisoned.new())

# Single remove (pass the Script class, not an instance)
entity.remove_component(C_Poisoned)

# Batch add — only ONE archetype move total
entity.add_components([C_Burning.new(), C_Wet.new()])

# Batch remove
entity.remove_components([C_Burning, C_Wet])
```

## Querying Components on an Entity

```gdscript
# Check presence
if entity.has_component(C_Health):
    pass

# Get instance (returns null if not found)
var hp: C_Health = entity.get_component(C_Health)

# Get all components as array
var all_comps = entity.get_components()
```

## Relationships

```gdscript
# Add
entity.add_relationship(Relationship.new(C_Likes.new(), other_entity))

# Remove
entity.remove_relationship(Relationship.new(C_Likes.new(), other_entity))

# Check
if entity.has_relationship(Relationship.new(C_Likes.new(), null)):  # wildcard
    pass
```

## Enable / Disable

```gdscript
entity.enabled = false   # excluded from queries by default
entity.enabled = true    # re-included
```

Queries can filter by enabled state using `.with_enabled()` or `.with_disabled()` on `QueryBuilder`.

## ecs_id

`ecs_id` is assigned by the World when the entity is first registered. It is stable for the entity's lifetime and is used in relationship slot keys and serialization.

```gdscript
print(entity.ecs_id)  # e.g. 42
```

## Rules

- Do **not** add or remove components from inside a system's `process()` while iterating — use `cmd` (CommandBuffer) instead.
- `define_components()` is called once during `_initialize()` before `on_ready()`.
- Batch operations (`add_components`, `remove_components`) cost only **one** archetype move.

## See Also

- [`gecs-component`](../gecs-component/SKILL.md) — Component API
- [`gecs-command-buffer`](../gecs-command-buffer/SKILL.md) — Safe structural changes
- [`gecs-relationship`](../gecs-relationship/SKILL.md) — Relationship API

