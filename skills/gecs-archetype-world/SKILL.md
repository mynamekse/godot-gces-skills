---
name: gecs-archetype-world
description: World and Archetype internals — hybrid archetype ECS storage, entity registration, archetype edges, cache invalidation, query pipeline, and relationship slot keys.
---

# GECS — World & Archetype Internals

## Overview

`World` (`world.gd`) is the central ECS manager. It uses a **hybrid archetype model**: entities with the same component signature share an `Archetype` object for cache-friendly iteration. `Archetype` (`archetype.gd`) is a compact, indexed entity store.

## World: Key Data Structures

| Field | Type | Description |
|---|---|---|
| `entities` | `Array[Entity]` | All registered entities. |
| `archetypes` | `Dictionary` | `int (signature hash) → Archetype` |
| `entity_to_archetype` | `Dictionary` | `Entity → Archetype` — fast per-entity lookup. |
| `_query_archetype_cache` | `Dictionary` | `int (cache_key) → Array[Archetype]` — caches query→archetype matches. |
| `cache_version` | `int` | Monotonically increments on every cache invalidation. |
| `_relation_type_archetype_index` | `Dictionary` | `String (relation_path) → {sig → Archetype}` — wildcard relationship index. |

## Archetype

An `Archetype` groups all entities with the **exact same component set**.

```
Archetype {
    signature: int               # hash of component_types
    component_types: Array       # sorted int keys + rel:// String keys
    entities: Array[Entity]      # flat array — O(n) iteration, great cache locality
    entity_to_index: Dictionary  # Entity → int (for O(1) swap-remove)
    enabled_bitset: PackedInt64Array  # 1 bit per entity, enabled state
    columns: Dictionary          # int (script_id) → Array[Component] (SoA)
    add_edges: Dictionary        # Variant → Archetype (O(1) transition cache)
    remove_edges: Dictionary     # Variant → Archetype
    neighbors: Dictionary        # source archetype id → Archetype (for cleanup)
}
```

## Entity Registration Flow

```
world.add_entity(entity)
  → _initialize() — replays all components through add_component() signals
  → _add_entity_to_archetype(entity)
      → _calculate_entity_signature(entity)   # QueryCacheKey.build(comp_scripts)
      → _get_or_create_archetype(sig, types)  # finds or creates Archetype
      → archetype.add_entity(entity)          # O(1) append + column update + bitset
      → entity_to_archetype[entity] = archetype
```

## Archetype Transitions (Component Add/Remove)

```
entity.add_component(C_Burning.new())
  → component_added signal → _on_entity_component_added()
      → _move_entity_to_new_archetype_fast(entity, old, comp_key, is_add=true)
          → old_archetype.get_add_edge(comp_key)  # O(1) edge cache hit
          → if no edge: _calculate_entity_signature() + _get_or_create_archetype()
          → old_archetype.set_add_edge(comp_key, new_archetype)  # cache for future
          → old_archetype.remove_entity(entity)   # O(1) swap-remove
          → new_archetype.add_entity(entity)      # O(1) append
          → entity_to_archetype[entity] = new_archetype
      → _invalidate_cache("entity_component_added")
```

## Query Pipeline

```
q.with_all([C_Health, C_Position]).execute()
  → QueryBuilder.execute()
      → _world._query(all, any, exclude, ...)
          → cache_key = QueryCacheKey.build(...)   # FNV hash
          → if _query_archetype_cache.has(cache_key): return cached archetypes
          → else: scan all archetypes → archetype.matches_query(_all, _any, _exclude)
          → cache result
          → flatten entity arrays from matching archetypes
```

## Cache Invalidation

```
_invalidate_cache(reason)
  → if _suppress_invalidation_depth > 0: mark _pending_invalidation and return
  → _query_archetype_cache.clear()
  → cache_version += 1
  → emit cache_invalidated signal
```

Batch operations (`add_entities`, `CommandBuffer.execute`) use `_begin_suppress()` / `_end_suppress()` to defer invalidation until after all mutations complete.

## Relationship Slot Keys

Each (relation, target) pair maps to a slot key string stored in the archetype:

```
rel://res://components/c_likes.gd::entity#42
rel://res://components/c_damaged.gd::comp://res://components/c_fire_damage.gd
rel://res://components/c_child_of.gd::script://res://entities/e_boss.gd
rel://res://components/c_likes.gd::*     (wildcard)
```

Slot keys are **part of the archetype signature** — each unique (relation, target) pair creates a **new archetype**. Avoid high-cardinality relationships (entity→entity with many unique targets).

## Archetype Edges (O(1) Transitions)

When a component is added/removed, the destination archetype is cached as an edge:
- `add_edges[comp_key]` → target archetype when adding this component.
- `remove_edges[comp_key]` → target archetype when removing this component.

On first transition: calculate signature + find/create archetype + cache edge. On subsequent transitions: O(1) edge lookup.

## Wildcard Relationship Index

`_relation_type_archetype_index[rel_path]` maps a relation script path to all archetypes that contain at least one slot key with that relation. Used to narrow candidates for wildcard queries (`Relationship.new(C_Likes.new(), null)`).

## Enabled State (Bitset)

Each `Archetype` stores a `PackedInt64Array` bitset — 1 bit per entity slot — tracking `enabled` state. This avoids doubling archetype count (old approach) and provides O(1) enabled/disabled filtering via `archetype.get_entities_by_enabled_state(enabled_only)`.

## Column Storage (SoA)

```
archetype.columns[script_instance_id] → Array[Component]
```

Enables Flecs-style cache-friendly iteration — iterate one component type as a contiguous array:

```gdscript
for archetype in q.with_all([C_Position, C_Velocity]).archetypes():
    var positions  = archetype.get_column(C_Position.get_instance_id())
    var velocities = archetype.get_column(C_Velocity.get_instance_id())
    for i in range(positions.size()):
        positions[i].x += velocities[i].dx * delta
```

## Observer Dispatch

```
component_added signal → _on_entity_component_added()
  → archetype move
  → _invalidate_cache()
  → component_added.emit(entity, component)
  → _handle_observer_component_added() → _dispatch_observer_event(ADDED, entity, comp)
      → lookup _obs_entries_by_event[ADDED]
      → filter by watched_paths, property filters, entity query match
      → entry.callable.call(event, entity, payload)
      → if PER_CALLBACK: obs.cmd.execute()
```

## World API (High Level)

```gdscript
world.add_entity(entity)
world.remove_entity(entity)
world.add_system(system)
world.remove_system(system)
world.add_observer(observer)
world.remove_observer(observer)
world.emit_event(&"event_name", entity, data)
world.process(delta, group)
world.serialize() → Dictionary
world.deserialize(data)
world.get_cache_stats() → Dictionary
world.group_entities_by_archetype(entities) → Dictionary
world.get_matching_archetypes(query_builder) → Array[Archetype]
```

## Rules

- Each unique component signature creates a new `Archetype` — minimize structural variation.
- Each unique (relation, target) pair **also** creates a new archetype — be careful with entity-to-entity relationships at scale.
- Do not access `world.archetypes` or `world.entity_to_archetype` directly in game code — use `QueryBuilder`.
- `cache_version` can be used by external tools to detect stale cached query results without signal subscription.

## See Also

- [`gecs-query`](../gecs-query/SKILL.md) — Query API + cache details
- [`gecs-relationship`](../gecs-relationship/SKILL.md) — Slot key patterns
- [`gecs-performance`](../gecs-performance/SKILL.md) — Column iteration, suppression

