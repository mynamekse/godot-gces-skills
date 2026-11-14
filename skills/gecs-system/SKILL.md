---
name: gecs-system
description: System — the logic unit in GECS. Covers process/process_all, groups, sub_systems, SystemTimer, parallel processing, CommandBuffer flush modes, and performance monitoring.
---

# GECS — System

## Overview

`System` (`system.gd`) extends `Node`. Systems contain all game logic. They query entities and iterate them every frame (or on a timer). Systems are grouped and ticked by the `World`.

## Source: `addons/gecs/ecs/system.gd`

```
class_name System
extends Node
```

## Key Exported Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `group` | `String` | `""` | Processing group. Empty = default group. |
| `active` | `bool` | `true` | If false, system is skipped entirely. |
| `paused` | `bool` | `false` | Runtime-only pause (skips processing). |
| `safe_iteration` | `bool` | `true` | Snapshots entity array before iterating (prevents mutation bugs). |
| `parallel_processing` | `bool` | `false` | Dispatch `process()` calls via WorkerThreadPool. |
| `tick_source` | `SystemTimer` | `null` | Only run when timer ticks. |
| `command_buffer_flush_mode` | `FlushMode` | `PER_SYSTEM` | When `cmd` auto-flushes. |

## FlushMode Enum

| Value | Meaning |
|---|---|
| `PER_SYSTEM` | Flush after all entities processed (default). |
| `PER_GROUP` | Flush after the entire group completes. |
| `MANUAL` | Never auto-flush; call `cmd.execute()` yourself. |

## Override: `query() → QueryBuilder`

Return the query that selects entities this system processes.

```gdscript
func query() -> QueryBuilder:
    return q.with_all([C_Health, C_Position])
```

## Override: `process(entity, components, delta)`

Called once per matching entity per frame.

```gdscript
func process(entity: Entity, components: Array, delta: float) -> void:
    var hp: C_Health = entity.get_component(C_Health)
    hp.current_hp -= 1 * delta
```

## Override: `process_all(entities, components, delta)`

Called once per frame with all matching entities.

```gdscript
func process_all(entities: Array[Entity], components: Array, delta: float) -> void:
    for entity in entities:
        # custom batch logic
        pass
```

> Implement either `process()` **or** `process_all()`, not both.

## Sub-Systems

Return multiple `[QueryBuilder, Callable]` tuples to run several queries in one system node. Optionally include a `SystemTimer` as a third element.

```gdscript
func sub_systems() -> Array[Array]:
    return [
        [q.with_all([C_Moving]),    _move,   null],
        [q.with_all([C_Rotating]),  _rotate, SystemTimer.new()],
    ]

func _move(entity: Entity, _comps: Array, delta: float) -> void:
    pass

func _rotate(entity: Entity, _comps: Array, delta: float) -> void:
    pass
```

## CommandBuffer (`cmd`)

Access `cmd` inside `process()` to queue structural changes safely:

```gdscript
func process(entity: Entity, components: Array, delta: float) -> void:
    var hp: C_Health = entity.get_component(C_Health)
    if hp.current_hp <= 0:
        cmd.remove_entity(entity)   # deferred — won't corrupt iteration
        cmd.add_component(entity, C_Dead.new())
```

## Performance Monitoring

When `performance_monitor = true` (and `ECS.debug = true`), the system registers Godot Performance monitors for min/avg/max execution time.

```gdscript
@export var performance_monitor: bool = true
```

## SystemTimer

Throttle a system to run every N seconds:

```gdscript
# In-editor: set tick_source to a SystemTimer resource
# In code:
var timer = SystemTimer.new()
timer.interval = 0.5
my_system.tick_source = timer
world.add_system(my_system)
```

See [`gecs-system-timer`](../gecs-system-timer/SKILL.md) for full API.

## Groups

```gdscript
# System runs only when ECS.process(delta, "physics") is called
@export var group: String = "physics"
```

## Rules

- Never modify `entities` array inside `process()` — use `cmd`.
- `safe_iteration = true` (default) snapshots the array; `false` is faster but risky.
- `parallel_processing = true` requires thread-safe process() implementations.
- `cmd` is lazy — created on first access.

## See Also

- [`gecs-query`](../gecs-query/SKILL.md) — Building queries
- [`gecs-command-buffer`](../gecs-command-buffer/SKILL.md) — Deferred mutations
- [`gecs-system-timer`](../gecs-system-timer/SKILL.md) — Timed systems

