---
name: gecs-query
description: QueryBuilder — the fluent API to select entities by components, relationships, groups, and properties. Covers with_all, with_any, without, with_relationship, execute, execute_one, archetypes, and caching.
---

# GECS — QueryBuilder

## Overview

`QueryBuilder` (`query_builder.gd`) is a **fluent builder** that constructs entity queries. Access it through `ECS.world.query` or the system/observer shortcut `q`.

## Source: `addons/gecs/ecs/query_builder.gd`

```
class_name QueryBuilder
extends RefCounted
```

## Component Filters

```gdscript
# Must have ALL of these
q.with_all([C_Health, C_Position])

# Must have AT LEAST ONE of these
q.with_any([C_Burning, C_Poisoned])

# Must have NONE of these
q.without([C_Dead, C_Frozen])
```

Filters can be chained:

```gdscript
q.with_all([C_Health]).with_any([C_Burning, C_Poisoned]).without([C_Dead])
```

## Property Queries (Component Value Filters)

Filter by property values inline using dictionaries:

```gdscript
# Health >= 50
q.with_all([{C_Health: {"current_hp": {"_gte": 50}}}])

# Operators: _eq, _neq, _gt, _gte, _lt, _lte, _in, _not_in
q.with_all([{C_Speed: {"value": {"_lt": 2.0}}}])

# Multiple properties
q.with_all([{C_Health: {"current_hp": {"_gt": 0}, "is_invincible": {"_eq": false}}}])
```

## Group Filters

```gdscript
q.with_all([C_Health]).in_group("enemies")
q.with_all([C_Health]).not_in_group("bosses")
```

## Enabled Filter

```gdscript
q.with_all([C_Health]).with_enabled()   # only enabled entities
q.with_all([C_Health]).with_disabled()  # only disabled entities
```

## Relationship Filters

```gdscript
# Entity has a "Likes" relationship pointing to alice
q.with_relationship([Relationship.new(C_Likes.new(), alice)])

# Wildcard: entity has ANY "Likes" relationship
q.with_relationship([Relationship.new(C_Likes.new(), null)])

# Exclude
q.without_relationship([Relationship.new(C_Enemy.new(), null)])
```

## Executing Queries

```gdscript
# Returns Array[Entity]
var entities = q.with_all([C_Health]).execute()

# Returns first match or null
var player = q.with_all([C_Player]).execute_one()
```

## Column-Based Iteration (High Performance)

Skip entity array flattening — iterate archetypes directly:

```gdscript
for archetype in q.with_all([C_Position, C_Velocity]).archetypes():
    var positions = archetype.get_column(C_Position.get_instance_id())
    var velocities = archetype.get_column(C_Velocity.get_instance_id())
    for i in range(positions.size()):
        positions[i].x += velocities[i].dx * delta
```

## Query Caching

Queries are cached by a structural hash key (`QueryCacheKey`). The cache is invalidated whenever archetypes change (component add/remove, entity add/remove).

```gdscript
# Cache key is built from:
# - all_components (sorted)
# - any_components (sorted)
# - exclude_components (sorted)
# - structural relationships (sorted)
# - groups (sorted)
```

**Cache invalidation triggers:**
- Entity added / removed from world
- Component added / removed from entity
- Relationship added / removed

Property-only changes do NOT invalidate the structural cache.

## Observer Event Modifiers

Used only inside `Observer.query()` or `Observer.sub_observers()`:

```gdscript
q.with_all([C_Health]).on_added()
q.with_all([C_Health]).on_removed()
q.with_all([C_Health]).on_changed()          # property change
q.with_all([C_Player]).on_match()            # entity newly matches query
q.with_all([C_Player]).on_unmatch()          # entity no longer matches
q.with_all([C_Enemy]).on_relationship_added()
q.with_all([C_Enemy]).on_relationship_removed()
q.with_all([C_Enemy]).on_event(&"damage_dealt")
```

## Rules

- `q` in systems/observers returns a **fresh** QueryBuilder every access — do not cache `q` itself.
- Property query dicts use the `_operator` convention: `{"_gte": 50}`.
- Structural queries (components, groups, structural relationships) are cached; post-filters (property queries) run after cache lookup.
- `execute()` returns a live array reference when only one archetype matches — do not mutate it directly.

## See Also

- [`gecs-observer`](../gecs-observer/SKILL.md) — Observer event modifiers
- [`gecs-relationship`](../gecs-relationship/SKILL.md) — Relationship queries
- [`gecs-archetype-world`](../gecs-archetype-world/SKILL.md) — How caching works internally

