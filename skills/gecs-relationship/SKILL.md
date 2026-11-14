---
name: gecs-relationship
description: Relationship — linking entities in GECS. Covers Relationship.new(), relation, target types (Entity/Component/Script), wildcard queries, property-query relationships, matches(), valid(), and archetype slot keys.
---

# GECS — Relationship

## Overview

`Relationship` (`relationship.gd`) extends `Resource`. Relationships link entities to other entities, components, or archetypes using a **(relation, target)** pair. Relations are `Component` instances that describe the link type; targets are the other side.

## Source: `addons/gecs/ecs/relationship.gd`

```
class_name Relationship
extends Resource
```

## Relationship Types

| Target Type | Use Case | Example |
|---|---|---|
| `Entity` | Link two specific entities | `Relationship.new(C_Likes.new(), alice)` |
| `Component` | Type hierarchy / data payload | `Relationship.new(C_Damaged.new(), C_FireDamage.new())` |
| `Script` | Archetype/type matching | `Relationship.new(C_ChildOf.new(), E_Boss)` |
| `null` | Wildcard (matches any target of that relation type) | `Relationship.new(C_Likes.new(), null)` |

## Creating Relationships

```gdscript
# Entity → Entity
var likes = Relationship.new(C_Likes.new(), e_alice)
e_bob.add_relationship(likes)

# Entity → Component (type hierarchy)
var fire_dmg = Relationship.new(C_Damaged.new(), C_FireDamage.new(50))
entity.add_relationship(fire_dmg)

# Entity → Archetype
var child_of = Relationship.new(C_ChildOf.new(), E_Boss)
e_minion.add_relationship(child_of)
```

## Querying Relationships

```gdscript
# Exact entity target
q.with_relationship([Relationship.new(C_Likes.new(), alice)])

# Wildcard: any target for this relation type
q.with_relationship([Relationship.new(C_Likes.new(), null)])

# Exclude
q.without_relationship([Relationship.new(C_Enemy.new(), null)])
```

## Property-Query Relationships

Filter by property values on the relation or target component:

```gdscript
# Filter by relation property
q.with_relationship([
    Relationship.new({C_Eats: {"value": {"_eq": 8}}}, e_apple)
])

# Filter by target component property
q.with_relationship([
    Relationship.new(C_Damage.new(), {C_Health: {"amount": {"_gte": 50}}})
])

# Filter both sides
q.with_relationship([
    Relationship.new(
        {C_Buff: {"duration": {"_gt": 10}}},
        {C_Player: {"level": {"_gte": 5}}}
    )
])
```

## matches(other) → bool

Checks if two relationships are equivalent (used internally by World/QueryBuilder).

```gdscript
var r1 = Relationship.new(C_Likes.new(), null)   # wildcard
var r2 = Relationship.new(C_Likes.new(), alice)
print(r1.matches(r2))  # true — wildcard matches any target
```

**Matching rules:**
- `null` relation or target = wildcard (matches anything).
- Component targets: match by script type (default) or by property query.
- Entity targets: match by reference equality.
- Script targets: match archetype — an entity instance matches its own script.

## valid() → bool

Returns true if the source is a valid entity and the target is valid (or null).

```gdscript
if not relationship.valid():
    entity.remove_relationship(relationship)
```

## Archetype Slot Keys

Each unique (relation_path, target_key) pair creates a **slot key** in the form:

```
rel://res://components/c_likes.gd::entity#42
rel://res://components/c_damaged.gd::comp://res://components/c_fire_damage.gd
rel://res://components/c_child_of.gd::script://res://entities/e_boss.gd
rel://res://components/c_likes.gd::*     (wildcard)
```

Slot keys are stored in the entity's archetype. Each unique (relation, target) pair creates a **new archetype** — be careful with high-cardinality entity-to-entity relationships (can cause archetype explosion).

## Rules

- Relation must be a **Component instance** (`C_Likes.new()`), not a class reference.
- Target must be an `Entity`, `Component`, `Script`, or `null`.
- Each unique entity target creates a new archetype — avoid hundreds of unique entity relationships.
- Property-query Relationships (`_is_query_relationship = true`) are **not** structural — they don't affect archetype placement.
- Remove relationships when the target entity is destroyed (World auto-cleans on `remove_entity`).

## See Also

- [`gecs-query`](../gecs-query/SKILL.md) — with_relationship() / without_relationship()
- [`gecs-observer`](../gecs-observer/SKILL.md) — RELATIONSHIP_ADDED / RELATIONSHIP_REMOVED events
- [`gecs-archetype-world`](../gecs-archetype-world/SKILL.md) — Slot key internals

