---
name: gecs-command-buffer
description: CommandBuffer — deferred structural change queue in GECS. Covers add/remove component/entity/relationship, add_custom, execute, flush modes, and safe iteration patterns.
---

# GECS — CommandBuffer

## Overview

`CommandBuffer` (`command_buffer.gd`) queues structural ECS operations (add/remove components, entities, relationships) for **deferred execution**. This ensures iteration is never interrupted by structural mutations.

## Source: `addons/gecs/ecs/command_buffer.gd`

```
class_name CommandBuffer
extends RefCounted
```

## Why Use CommandBuffer?

Directly mutating entities inside a system's `process()` while iterating them causes bugs:
- Skipped entities (iteration index shifts)
- Crashes (array mutated during iteration)
- Observer re-entrancy issues

`cmd` queues the operations and executes them **after** the system completes.

## Accessing cmd

In a **System**:
```gdscript
# cmd is a lazy property on System — just use it
cmd.remove_entity(entity)
```

In an **Observer**:
```gdscript
cmd.add_component(entity, C_Damaged.new())
```

## Available Commands

```gdscript
# Entity management
cmd.add_entity(entity)
cmd.remove_entity(entity)

# Component management
cmd.add_component(entity, C_Burning.new())
cmd.remove_component(entity, C_Burning)

# Batch component operations (single archetype move)
cmd.add_components(entity, [C_Burning.new(), C_Wet.new()])
cmd.remove_components(entity, [C_Burning, C_Wet])

# Relationship management
cmd.add_relationship(entity, Relationship.new(C_Likes.new(), other))
cmd.remove_relationship(entity, Relationship.new(C_Likes.new(), other))

# Custom operation (arbitrary callable)
cmd.add_custom(func(): do_something_complex())
```

## execute()

Runs all queued commands in order. Called automatically based on `FlushMode`.

```gdscript
cmd.execute()   # manual flush
```

**Internally:**
1. Swaps `_commands` to a local copy (re-entrant safe).
2. Calls `_world._begin_suppress()` — batches cache invalidation.
3. Runs all callables.
4. Calls `_world._end_suppress()` — fires one deferred cache invalidation.

## Flush Modes

### System.FlushMode

| Value | Description |
|---|---|
| `PER_SYSTEM` (default) | Flush after the system finishes all entities. |
| `PER_GROUP` | Flush after the entire system group completes. |
| `MANUAL` | Never auto-flush; you call `cmd.execute()` yourself. |

### Observer.FlushMode

| Value | Description |
|---|---|
| `PER_CALLBACK` (default) | Flush after each `each()` callback. |
| `MANUAL` | Never auto-flush. |

## Statistics

```gdscript
var stats = cmd.get_stats()
# {
#   "commands_queued": 10,
#   "commands_executed": 10,
#   "last_execution_time_ms": 0.03
# }
```

## Utility

```gdscript
cmd.clear()          # discard all queued commands without running them
cmd.is_empty()       # true if nothing queued
cmd.size()           # number of queued commands
```

## Rules

- Commands execute in **FIFO order** (order they were queued).
- Each command captures the entity reference in a lambda — `is_instance_valid(entity)` is checked before execution.
- Commands queued **during** `execute()` land in a fresh `_commands` and are flushed by the next `execute()` call (not recursively in the same flush).
- `cmd` is **lazy** — created on first access; no overhead if never used.
- Prefer batch commands (`add_components`, `remove_components`) over multiple single operations.

## See Also

- [`gecs-system`](../gecs-system/SKILL.md) — FlushMode on System
- [`gecs-observer`](../gecs-observer/SKILL.md) — FlushMode on Observer

