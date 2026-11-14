---
name: gecs-component
description: Component — pure data Resource in GECS. Covers class definition, @export properties, property_changed signal, parent reference, and serialize().
---

# GECS — Component

## Overview

`Component` (`component.gd`) extends `Resource`. Components are **pure data containers** — no logic. They attach to entities and drive system behaviour through queries.

## Source: `addons/gecs/ecs/component.gd`

```
class_name Component
extends Resource
```

## Defining a Component

```gdscript
# components/c_health.gd
class_name C_Health
extends Component

@export var max_hp: int = 100
@export var current_hp: int = 100
@export var is_invincible: bool = false
```

Only `@export` properties are included in `serialize()`. Non-exported vars are transient.

## Key Properties

| Property | Type | Description |
|---|---|---|
| `parent` | `Entity` | Back-reference to the owning entity. Set by World when component is added. |

## Signals

| Signal | Args | When |
|---|---|---|
| `property_changed` | `component, property_name, old_value, new_value` | Emitted manually from setters to trigger `on_changed` observers. |

### Emitting property_changed for Observer Support

```gdscript
class_name C_Health
extends Component

@export var current_hp: int = 100:
    set(val):
        var old = current_hp
        current_hp = val
        if old != val:
            property_changed.emit(self, "current_hp", old, val)
```

> **Important:** Observers only fire for `CHANGED` events when `property_changed` is explicitly emitted. Direct property assignment without a setter does NOT trigger observers automatically — this is intentional for performance.

## serialize()

Returns a `Dictionary` of all `@export` property names → values. Used by `world.serialize()` for save/load.

```gdscript
var hp_comp = C_Health.new()
hp_comp.current_hp = 75
print(hp_comp.serialize())
# { "max_hp": 100, "current_hp": 75, "is_invincible": false }
```

## Component Identity

Components are identified by their **Script instance_id** (not resource path, not class name). This is stable per loaded script for the lifetime of the process.

```gdscript
var key = C_Health.get_instance_id()  # int used in archetype + query cache
```

## Best Practices

- Keep components **data-only**. No game logic, no `_process`, no signals besides `property_changed`.
- Use `@export` for all data you want persisted or inspector-editable.
- Use typed properties for performance.
- Avoid storing node references in components — store `ecs_id` or use Relationships instead.

## Rules

- Do not store mutable references to other entities as plain vars — use relationships.
- Property setters that emit `property_changed` must emit **after** the value is set.
- `parent` is set by the World; do not set it manually.

## See Also

- [`gecs-entity`](../gecs-entity/SKILL.md) — Attaching components to entities
- [`gecs-observer`](../gecs-observer/SKILL.md) — Reacting to property changes
- [`gecs-serialization`](../gecs-serialization/SKILL.md) — Save/load

