---
name: gecs-serialization
description: GECS serialization — save/load world state. Covers ECS.save/load, world.serialize/deserialize, Component.serialize(), @export requirement, entity preprocessors/postprocessors.
---

# GECS — Serialization

## Overview

GECS provides built-in world serialization through `ECS.save()` / `ECS.load_save()` and `World.serialize()` / `World.deserialize()`. Only `@export` component properties are saved.

## API

### High-Level (via ECS singleton)

```gdscript
# Save
ECS.save("user://save.json")

# Load
ECS.load_save("user://save.json")

# Manual snapshot
var snapshot: Dictionary = ECS.serialize()

# Manual restore
ECS.deserialize(snapshot)
```

### Low-Level (via World)

```gdscript
var data = ECS.world.serialize()   # returns Dictionary

ECS.world.deserialize(data)        # restores from Dictionary
```

## What Gets Serialized

- All entities registered with the world.
- All `@export` properties on each `Component` attached to those entities.
- Relationships are serialized as `(relation_script_path, target_info)` pairs.

## Component Requirements

Only `@export` properties are included:

```gdscript
class_name C_Health
extends Component

@export var max_hp: int = 100       # ✓ serialized
@export var current_hp: int = 100   # ✓ serialized
var runtime_flag: bool = false      # ✗ NOT serialized (not exported)
```

```gdscript
# Manual check: serialize() returns only @export vars
var hp = C_Health.new()
hp.current_hp = 42
print(hp.serialize())
# { "max_hp": 100, "current_hp": 42 }
```

## Entity Preprocessors / Postprocessors

Hook into serialization with global callables on `ECS`:

```gdscript
# Called BEFORE each entity is serialized
ECS.entity_preprocessors.append(func(entity: Entity):
    # e.g. mark entity as dirty, snapshot AI state
    pass
)

# Called AFTER each entity is deserialized / restored
ECS.entity_postprocessors.append(func(entity: Entity):
    # e.g. reconnect node references, re-subscribe to signals
    pass
)
```

## Deserialization Flow

1. World clears all existing entities.
2. For each entity in `data`:
   - Re-creates the entity node (using stored script path).
   - Restores `@export` properties on each component.
   - Re-adds the entity to the world.
3. Entity `postprocessors` run on each restored entity.

## Save File Format

```json
{
  "entities": [
    {
      "id": "...",
      "script": "res://entities/e_player.gd",
      "components": {
        "res://components/c_health.gd": { "max_hp": 100, "current_hp": 75 },
        "res://components/c_position.gd": { "x": 10.5, "y": 3.0 }
      },
      "relationships": [...]
    }
  ]
}
```

## Rules

- Only `@export` variables are saved — transient state must be rebuilt via postprocessors.
- Entity scene nodes that are not registered with `world.add_entity()` are not serialized.
- Component scripts must have stable `resource_path` values — do not rename scripts after saving.
- Call `ECS.load_save()` only after `ECS.world` is set.
- Deserialization clears the world first — ensure observers and systems are re-registered if they depend on world events.

## Best Practices

- Keep serializable state in `@export` vars; keep transient state in non-exported vars.
- Use postprocessors to restore node references, timer states, and cached lookups.
- Test save/load with `ECS.debug = true` to catch assertion failures during restore.

## See Also

- [`gecs-component`](../gecs-component/SKILL.md) — serialize() and @export
- [`gecs-ecs-singleton`](../gecs-ecs-singleton/SKILL.md) — ECS.save/load, preprocessors

