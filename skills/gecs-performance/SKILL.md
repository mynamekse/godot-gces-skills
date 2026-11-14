---
name: gecs-performance
description: GECS performance optimization guide. Covers column-based iteration, batch operations, cache suppression, archetype explosion avoidance, parallel processing, SystemTimer, and profiling.
---

# GECS — Performance Optimization

## Overview

GECS is a hybrid archetype ECS optimized for Godot 4. Its key performance features are: archetype-level query caching, column (SoA) storage, O(1) entity add/remove via swap-remove, edge-cached archetype transitions, and bitset-based enabled filtering.

## 1. Column-Based Iteration (Highest Impact)

Instead of `execute()` → iterate entities → `get_component()` per entity, use `archetypes()` for Flecs-style direct array access:

```gdscript
# SLOW — O(n) dictionary lookup per entity
var entities = q.with_all([C_Position, C_Velocity]).execute()
for entity in entities:
    var pos = entity.get_component(C_Position)
    var vel = entity.get_component(C_Velocity)
    pos.x += vel.dx * delta

# FAST — O(1) column access, excellent cache locality
for archetype in q.with_all([C_Position, C_Velocity]).archetypes():
    var positions  = archetype.get_column(C_Position.get_instance_id())
    var velocities = archetype.get_column(C_Velocity.get_instance_id())
    for i in range(positions.size()):
        positions[i].x += velocities[i].dx * delta
```

## 2. Batch Component Operations

Each `add_component()` / `remove_component()` triggers an archetype move + cache invalidation. Batch operations cost only **one** move:

```gdscript
# SLOW — 3 archetype moves
entity.add_component(C_Burning.new())
entity.add_component(C_Wet.new())
entity.add_component(C_Frozen.new())

# FAST — 1 archetype move
entity.add_components([C_Burning.new(), C_Wet.new(), C_Frozen.new()])
```

Same for `cmd`:
```gdscript
cmd.add_components(entity, [C_A.new(), C_B.new()])   # 1 move
cmd.remove_components(entity, [C_A, C_B])             # 1 move
```

## 3. CommandBuffer for Bulk Mutations

`CommandBuffer.execute()` wraps all commands in `_begin_suppress()` / `_end_suppress()` — cache is invalidated **once** at the end instead of once per command.

```gdscript
# System FlushMode.PER_SYSTEM (default) — good
# System FlushMode.PER_GROUP — better for systems that do lots of mutations
# System FlushMode.MANUAL — best when you control exactly when to flush
```

## 4. Use safe_iteration = false Carefully

`safe_iteration = true` (default) snapshots the entity array before iteration (O(N) copy). Set to `false` if your `process()` never mutates the entity list:

```gdscript
@export var safe_iteration: bool = false   # only if no add/remove in process()
```

## 5. Throttle with SystemTimer

Don't run expensive systems every frame:

```gdscript
func _ready():
    tick_source = SystemTimer.new()
    tick_source.interval = 0.5   # 2 Hz instead of 60 Hz
```

## 6. Parallel Processing

Enable for CPU-heavy systems with no shared mutable state:

```gdscript
@export var parallel_processing: bool = true
# process() must be thread-safe — no scene tree calls, no signals
```

## 7. Avoid Archetype Explosion

Each unique combination of (components + relationships) creates a new archetype. Excessive archetypes slow down query scans.

**Causes:**
- Unique entity-to-entity relationships (e.g. 1000 entities each pointing to a different target → 1000 archetypes).
- Dynamic component sets that vary per entity.

**Solutions:**
- Use component instances as relationship targets (type matching) instead of entity instances.
- Share archetypes: entities with the same logical "type" should have the same component set.
- Monitor: `ECS.world.get_cache_stats()["total_archetypes"]` — >500 triggers a warning in debug mode.

```gdscript
# BAD — creates N archetypes (one per unique target entity)
for entity in enemies:
    entity.add_relationship(Relationship.new(C_Target.new(), player))

# BETTER — use entity ecs_id in a component instead
var hp: C_Target = C_Target.new()
hp.target_id = player.ecs_id
entity.add_component(hp)
```

## 8. Query Cache Efficiency

Structural queries are cached. Keep queries consistent across frames to maximize cache hits.

```gdscript
# Check cache stats at runtime
var stats = ECS.world.get_cache_stats()
print("Hit rate: ", stats.hit_rate)
print("Total archetypes: ", stats.total_archetypes)
print("Invalidations: ", stats.invalidation_count)
```

## 9. Enabled Filter via Bitset

Using `.with_enabled()` / `.with_disabled()` is O(n/64) via bitset — faster than checking `entity.enabled` manually in a loop.

```gdscript
var active = q.with_all([C_Unit]).with_enabled().execute()
```

## 10. Performance Monitoring

Enable per-system Godot Performance monitors:

```gdscript
@export var performance_monitor: bool = true
```

Access in Godot Monitor panel:
- `GECS/SystemName/min_ms`
- `GECS/SystemName/avg_ms`
- `GECS/SystemName/max_ms`

## 11. sub_systems() for Multiple Queries

Group related queries under one System node rather than creating many separate nodes:

```gdscript
func sub_systems() -> Array[Array]:
    return [
        [q.with_all([C_Moving]),   _move,   null],
        [q.with_all([C_Rotating]), _rotate, null],
    ]
```

## Summary Table

| Technique | Impact | Notes |
|---|---|---|
| Column iteration (`archetypes()`) | ★★★ | Best cache locality |
| Batch add/remove components | ★★★ | 1 archetype move vs N |
| CommandBuffer flush control | ★★ | 1 cache invalidation vs N |
| `safe_iteration = false` | ★ | Small — avoids O(N) snapshot |
| `parallel_processing = true` | ★★ | CPU-bound only, thread-safe code |
| `SystemTimer` throttling | ★★★ | Huge for AI/pathfinding |
| Avoid archetype explosion | ★★★ | Keep <100 archetypes |
| `with_enabled()` bitset filter | ★ | Faster than manual check |

## See Also

- [`gecs-archetype-world`](../gecs-archetype-world/SKILL.md) — Internals of archetype storage
- [`gecs-system`](../gecs-system/SKILL.md) — parallel_processing, FlushMode
- [`gecs-system-timer`](../gecs-system-timer/SKILL.md) — Throttling systems
- [`gecs-command-buffer`](../gecs-command-buffer/SKILL.md) — Batch mutation

