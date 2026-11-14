---
name: gecs-observer
description: Observer — reactive event-driven node in GECS. Covers Event enum, FlushMode, query(), each(), sub_observers(), yield_existing, monitor (on_match/on_unmatch), and emit_event.
---

# GECS — Observer

## Overview

`Observer` (`observer.gd`) extends `Node`. Observers are **reactive** — they fire callbacks when ECS events occur (component added/removed/changed, entity matches/unmatches a query, relationships change, or custom events).

## Source: `addons/gecs/ecs/observer.gd`

```
class_name Observer
extends Node
```

## Event Enum

| Value | Description |
|---|---|
| `ADDED` | A watched component was added to a matching entity. |
| `REMOVED` | A watched component was removed from a matching entity. |
| `CHANGED` | A watched property emitted `property_changed` on a matching entity. |
| `MATCH` | Entity newly satisfies the query (monitor mode). |
| `UNMATCH` | Entity no longer satisfies the query (monitor mode). |
| `RELATIONSHIP_ADDED` | A relationship was added to a matching entity. |
| `RELATIONSHIP_REMOVED` | A relationship was removed from a matching entity. |

## FlushMode Enum

| Value | Description |
|---|---|
| `PER_CALLBACK` | Flush `cmd` after every `each()` call (default, safest). |
| `MANUAL` | Flush only on explicit `cmd.execute()` / `world.flush_command_buffers()`. |

## Key Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `active` | `bool` | `true` | Skipped entirely when false. |
| `paused` | `bool` | `false` | Runtime-only pause. |
| `yield_existing` | `bool` | `false` | Fire retroactively for pre-existing entities on registration. |
| `command_buffer_flush_mode` | `FlushMode` | `PER_CALLBACK` | When `cmd` flushes. |
| `q` | `QueryBuilder` | — | Fresh QueryBuilder each access. |
| `cmd` | `CommandBuffer` | lazy | Deferred structural change buffer. |

## Override: `query() → QueryBuilder`

Return a QueryBuilder with **event modifiers** to declare what this observer watches.

```gdscript
func query() -> QueryBuilder:
    return q.with_all([C_Health]).on_added().on_removed()
```

> **Warning:** If `query()` returns a QueryBuilder with no event modifiers, the observer will silently never fire. A `push_warning` is emitted in editor builds.

## Override: `each(event, entity, payload)`

The unified callback. Fires for every event declared in `query()`.

```gdscript
func each(event: Variant, entity: Entity, payload: Variant = null) -> void:
    match event:
        Observer.Event.ADDED:
            print("Health added to ", entity.name, " — comp: ", payload)
        Observer.Event.REMOVED:
            print("Health removed from ", entity.name)
        Observer.Event.CHANGED:
            print("Property changed: ", payload.property, " = ", payload.new_value)
```

### Payload Shape by Event

| Event | `payload` type | Contents |
|---|---|---|
| `ADDED` | `Component` | The added component instance. |
| `REMOVED` | `Component` | The removed component instance (detached). |
| `CHANGED` | `Dictionary` | `{component, property, new_value, old_value}` |
| `MATCH` | `null` | — |
| `UNMATCH` | `null` | — |
| `RELATIONSHIP_ADDED` | `Relationship` | The added relationship. |
| `RELATIONSHIP_REMOVED` | `Relationship` | The removed relationship. |
| Custom | `Variant` | Whatever was passed to `emit_event()`. |

## Override: `sub_observers() → Array[Array]`

Multiple reactive axes in one observer node. Each tuple: `[QueryBuilder, Callable, yield_existing_override?]`.

```gdscript
func sub_observers() -> Array[Array]:
    return [
        [q.with_all([C_Health]).on_added(),   _on_health_added],
        [q.with_all([C_Health]).on_removed(),  _on_health_removed],
        [q.with_all([C_Player]).on_match(),    _on_player_match, true],
    ]

func _on_health_added(event, entity, payload):
    print("Health granted to ", entity.name)

func _on_health_removed(event, entity, payload):
    print("Health removed from ", entity.name)

func _on_player_match(event, entity, payload):
    print("Player now matches: ", entity.name)
```

## Monitor Mode (on_match / on_unmatch)

Monitors track entity membership continuously — they fire MATCH when an entity first satisfies the query and UNMATCH when it no longer does.

```gdscript
func query() -> QueryBuilder:
    return q.with_all([C_Player, C_Alive]).on_match().on_unmatch()

func each(event, entity, payload):
    match event:
        Observer.Event.MATCH:   GameUI.show_player_hud(entity)
        Observer.Event.UNMATCH: GameUI.hide_player_hud(entity)
```

## Changed Property Filter

```gdscript
# Only fire when "current_hp" property changes
func query() -> QueryBuilder:
    return q.with_all([C_Health]).on_changed([&"current_hp"])
```

## Relationship Events

```gdscript
func query() -> QueryBuilder:
    return q.with_all([C_Unit]).on_relationship_added(C_Enemy)
           .on_relationship_removed(C_Enemy)
```

## Custom Events

```gdscript
# Emitting:
ECS.world.emit_event(&"damage_dealt", target_entity, {"amount": 10})

# Listening:
func query() -> QueryBuilder:
    return q.with_all([C_Player]).on_event(&"damage_dealt")

func each(event, entity, payload):
    print("Damage: ", payload.amount)
```

## yield_existing

When `true`, the observer fires `ADDED` (or `MATCH`) retroactively for all entities that already match the query at registration time:

```gdscript
@export var yield_existing: bool = true
```

## setup()

Override for one-shot initialization after the observer is added to the World:

```gdscript
func setup() -> void:
    print("Observer ready, world=", _world)
```

## Rules

- Always chain at least one event modifier (`.on_added()`, `.on_match()`, etc.) to `query()`.
- Property changes only fire CHANGED observers if the component emits `property_changed`.
- Observers are NOT ticked by time — to throttle, use `FlushMode.MANUAL` and flush from a timed System.
- Use `cmd` for structural changes inside `each()` to avoid re-entrant world mutation.

## See Also

- [`gecs-query`](../gecs-query/SKILL.md) — QueryBuilder event modifiers
- [`gecs-component`](../gecs-component/SKILL.md) — property_changed signal
- [`gecs-command-buffer`](../gecs-command-buffer/SKILL.md) — Safe mutations in observers

