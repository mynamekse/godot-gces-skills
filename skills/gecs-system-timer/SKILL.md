---
name: gecs-system-timer
description: SystemTimer — throttles system execution to a fixed interval. Covers interval, single_shot, active, ticked, advance(), reset(), and sharing timers across systems.
---

# GECS — SystemTimer

## Overview

`SystemTimer` (`system_timer.gd`) is a tick-rate controller for Systems. Assign a `SystemTimer` to `System.tick_source` to run a system every N seconds instead of every frame.

## Source: `addons/gecs/ecs/system_timer.gd`

```
class_name SystemTimer
extends RefCounted
```

## Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `interval` | `float` | `1.0` | Seconds between ticks. Must be > 0. |
| `single_shot` | `bool` | `false` | If true, fires once then sets `active = false`. |
| `active` | `bool` | `true` | Inactive timers never tick. |
| `time_elapsed` | `float` | `0.0` | Accumulated time since last tick (carries overshoot to prevent drift). |
| `ticked` | `bool` | `false` | True only on the frame the timer ticked. |
| `tick_count` | `int` | `0` | Total ticks since creation or last `reset()`. |

## advance(delta)

Called once per group by `World` before processing systems. Sets `ticked = true` when `time_elapsed >= interval`.

```gdscript
# Internally (World calls this):
timer.advance(delta)
if timer.ticked:
    system.process_entities(delta)
```

## reset()

Resets `time_elapsed`, `tick_count`, `ticked` to 0 and sets `active = true`.

```gdscript
timer.reset()
```

## Usage Patterns

### Basic interval timer

```gdscript
class_name S_AIThink
extends System

func _ready():
    tick_source = SystemTimer.new()
    tick_source.interval = 0.5  # run every 500ms

func query() -> QueryBuilder:
    return q.with_all([C_AI, C_Position])

func process(entity: Entity, components: Array, delta: float) -> void:
    # Called at most every 500ms
    pass
```

### One-shot timer (intro sequence)

```gdscript
var intro_timer = SystemTimer.new()
intro_timer.interval = 3.0
intro_timer.single_shot = true
s_intro.tick_source = intro_timer
world.add_system(s_intro)
# System runs once after 3 seconds, then never again
```

### Shared timer (synchronized systems)

Multiple systems can share the same `SystemTimer` instance to guarantee they all run on the **exact same frame**:

```gdscript
var sync_timer = SystemTimer.new()
sync_timer.interval = 0.25  # 4 times per second

s_pathfind.tick_source = sync_timer
s_animation_update.tick_source = sync_timer
# Both run together every 250ms
```

> **Tip:** World deduplicates shared timer instances per group — `advance()` is called only once per unique timer even if N systems reference it.

## Sub-system timers

```gdscript
func sub_systems() -> Array[Array]:
    var slow_timer = SystemTimer.new()
    slow_timer.interval = 2.0
    return [
        [q.with_all([C_Fast]).execute(), _fast_process, null],
        [q.with_all([C_Slow]).execute(), _slow_process, slow_timer],
    ]
```

## Rules

- `interval` must be > 0.
- Overshoot is carried over — `time_elapsed -= interval` after each tick (prevents timer drift).
- Sharing timers guarantees synchronized execution on the same frame.
- Setting `active = false` stops the timer without resetting `time_elapsed`.

## See Also

- [`gecs-system`](../gecs-system/SKILL.md) — `tick_source` and `sub_systems()`

