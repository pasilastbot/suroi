# Rendering — Object Lifecycle Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/rendering/README.md -->
<!-- @source: client/src/scripts/game.ts, client/src/scripts/objects/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents how client-side game objects are created, updated, and destroyed from `UpdatePacket` data. Objects are mapped by category to PixiJS `Container` subclasses.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/game.ts | `processUpdatePacket()` — full/partial/deleted object handling | High |
| @file client/src/scripts/objects/gameObject.ts | `GameObject` base — `updateFromData()`, interpolation, destroy | High |
| @file client/src/scripts/objects/player.ts | Player rendering, weapon, emotes | High |

## ObjectClassMapping

| Category | Class |
|----------|-------|
| Player | `Player` |
| Obstacle | `Obstacle` |
| DeathMarker | `DeathMarker` |
| Loot | `Loot` |
| Building | `Building` |
| Decal | `Decal` |
| Parachute | `Parachute` |
| Projectile | `Projectile` |
| SyncedParticle | `SyncedParticle` |

## UpdatePacket Processing Order

1. **fullDirtyObjects** — Create or full-update
2. **partialDirtyObjects** — Delta update (object must exist)
3. **deletedObjects** — Destroy and remove from `objects`
4. **deserializedBullets** — Add to `bullets` (client-side only)
5. **explosions** — Spawn explosion effects
6. **emotes** — Show emote on player
7. **GasManager** — Update gas zone
8. **UI** — aliveCount, etc.

## Create / Update / Destroy Flow

```
fullDirtyObjects entry { id, type, data }:
    if object undefined or destroyed:
        → new ObjectClassMapping[type](id, data)
        → objects.add(object)
    else:
        → object.updateFromData(data, false)

partialDirtyObjects entry { id, data }:
    → object = objects.get(id)
    → object.updateFromData(data, false)

deletedObjects entry id:
    → object.destroy()
    → objects.delete(object)
```

## Business Rules

- Bullets and explosions are client-only effects; they are not in `objects`
- `updateFromData(data, isNew)` — `isNew` true when constructor calls it
- Position/rotation changes trigger interpolation; `updateContainerPosition()` / `updateContainerRotation()` run on PixiJS ticker

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Rendering overview
- **Tier 2:** [../packets/](../../packets/) — UpdatePacket structure
- **Tier 3:** [../objects/modules/client-objects.md](../../objects/modules/client-objects.md) — Client object implementations
