# Game Data Model

<!-- @tier: 1 -->
<!-- @see-also: docs/subsystems/definitions/, docs/subsystems/objects/ -->
<!-- @source: common/src/constants.ts, common/src/utils/objectDefinitions.ts, common/src/definitions/ -->
<!-- @updated: 2026-03-04 -->

## Overview

All entities in Suroi are modeled through two orthogonal systems:

1. **Definitions** — Static, read-only typed objects that describe what something *is* (e.g., what an AK-47 does, how a barrel looks).
2. **Game Objects** — Runtime instances tracked on the server and client, representing what something *is doing* right now (position, health, etc.).

Both systems are rooted in `common/` so that clients and servers share the same vocabulary.

## Object Categories

Every runtime object belongs to exactly one `ObjectCategory`. The category determines how the object is serialized, rendered, and simulated.

```typescript
// common/src/constants.ts
export enum ObjectCategory {
    Player,         // 0 — Human player or bot
    Obstacle,       // 1 — Destructible/solid world object
    DeathMarker,    // 2 — Visual marker at a player's death location
    Loot,           // 3 — Dropped item in the world
    Building,       // 4 — Multi-part structure (walls, floors, ceilings)
    Decal,          // 5 — Non-interactive ground texture
    Parachute,      // 6 — Player's entry parachute
    Projectile,     // 7 — Thrown projectile (grenade, etc.)
    SyncedParticle  // 8 — Server-authoritative particle effect
}
```

Object IDs are 16-bit unsigned integers (0–65535), allocated per game by `IDAllocator`.

## Definition Types

All *definitions* (static game data) carry a `DefinitionType` tag. The `ObjectDefinitions<Def>` class manages any definition list, providing lookup by `idString` (string key) or numeric index.

```typescript
// common/src/utils/objectDefinitions.ts
export enum DefinitionType {
    Ammo, Armor, Backpack, Badge, Building, Bullet,
    Decal, Emote, Explosion, Gun, HealingItem,
    MapPing, MapIndicator, Melee, Obstacle, Perk,
    Scope, Skin, SyncedParticle, Throwable
}
```

### Definition Collections

| Collection | File | Definition Type |
|------------|------|-----------------|
| `Guns` | `definitions/items/guns.ts` | `Gun` |
| `Melees` | `definitions/items/melees.ts` | `Melee` |
| `Throwables` | `definitions/items/throwables.ts` | `Throwable` |
| `Ammos` | `definitions/items/ammos.ts` | `Ammo` |
| `HealingItems` | `definitions/items/healingItems.ts` | `HealingItem` |
| `Armors` | `definitions/items/armors.ts` | `Armor` |
| `Backpacks` | `definitions/items/backpacks.ts` | `Backpack` |
| `Scopes` | `definitions/items/scopes.ts` | `Scope` |
| `Perks` | `definitions/items/perks.ts` | `Perk` |
| `Skins` | `definitions/items/skins.ts` | `Skin` |
| `Obstacles` | `definitions/obstacles.ts` | `Obstacle` |
| `Buildings` | `definitions/buildings.ts` | `Building` |
| `Bullets` | `definitions/bullets.ts` | `Bullet` |
| `Decals` | `definitions/decals.ts` | `Decal` |
| `Explosions` | `definitions/explosions.ts` | `Explosion` |
| `Emotes` | `definitions/emotes.ts` | `Emote` |
| `Loots` | `definitions/loots.ts` | (union of item types) |
| `MapPings` | `definitions/mapPings.ts` | `MapPing` |
| `SyncedParticles` | `definitions/syncedParticles.ts` | `SyncedParticle` |
| `Badges` | `definitions/badges.ts` | `Badge` |

### Serialization Size

If a definition list has ≤ 255 entries, its index is written as **1 byte** (`uint8`). If it exceeds 255, the index becomes **2 bytes** (`uint16`). This is tracked by `ObjectDefinitions.overLength`.

## Player Model

The `Player` is the most complex runtime entity.

### Stats

| Constant | Value | Location |
|----------|-------|----------|
| Default health | 200 HP | `GameConstants.player.defaultHealth` |
| Max adrenaline | 100 | `GameConstants.player.maxAdrenaline` |
| Max shield | 100 | `GameConstants.player.maxShield` |
| Max infection | 100 | `GameConstants.player.maxInfection` |
| Base speed | 0.06 u/ms | `GameConstants.player.baseSpeed` |
| Collision radius | 2.25 units | `GameConstants.player.radius` |
| Max mouse distance | 256 units | `GameConstants.player.maxMouseDist` |
| Name max length | 16 chars | `GameConstants.player.nameMaxLength` |

### Inventory Slots

```
Slot 0 — Gun (primary)
Slot 1 — Gun (secondary)
Slot 2 — Melee
Slot 3 — Throwable
```

Players can carry at most 1 active perk (`maxPerkCount = 1`) with up to 4 perk slots.

### Player Modifiers

Items can modify player attributes via `WearerAttributes`. All modifiers are multiplicative (default 1.0) except additive ones (e.g. `hpRegen`, `minAdrenaline`):

| Modifier | Type | Default |
|----------|------|---------|
| `maxHealth` | multiplier | 1.0 |
| `maxAdrenaline` | multiplier | 1.0 |
| `baseSpeed` | multiplier | 1.0 |
| `size` | multiplier | 1.0 |
| `reload` | multiplier | 1.0 |
| `fireRate` | multiplier | 1.0 |
| `adrenDrain` | multiplier | 1.0 |
| `hpRegen` | additive | 0 HP/tick |
| `minAdrenaline` | additive | 0 |

`EventModifiers` can additionally fire on `kill` or `damageDealt` events.

## Layer System

The world uses a vertical layer system to support multi-floor buildings:

| Layer | Value | Description |
|-------|-------|-------------|
| `Basement` | -2 | Underground level |
| `ToBasement` | -1 | Transition to basement |
| `Ground` | 0 | Default street level |
| `ToUpstairs` | 1 | Transition to upper floor |
| `Upstairs` | 2 | Upper floor |

Collision layers determine which objects interact:

| Mode | Behavior |
|------|----------|
| `Layers.All` | Collide with objects on all layers |
| `Layers.Adjacent` | Collide with same or adjacent layers |
| `Layers.Equal` | Only collide with the same layer |

## Z-Index (Render Order)

Client-side render order is controlled by `ZIndexes` (used as PixiJS `zIndex`):

```
Ground → BuildingsFloor → Decals → DeadObstacles → DeathMarkers →
Explosions → ObstaclesLayer1 → Loot → GroundedThrowables →
ObstaclesLayer2 → TeammateName → Bullets → DownedPlayers → Players →
ObstaclesLayer3 (bushes) → AirborneThrowables → ObstaclesLayer4 (trees) →
BuildingsCeiling → ObstaclesLayer5 → Emotes → Gas
```

## World Coordinates

| Constant | Value |
|----------|-------|
| Grid cell size | 32 units |
| Max world position | 1924 units (≈ 60×60 grid cells) |
| Object scale range | 0.15 – 3.0 |

Positions are serialized as 2-byte fixed-point values (0–65535 mapped to 0–1924 units).

## Gas Model

The gas zone has three states:

| State | Description |
|-------|-------------|
| `Inactive` | Gas not yet active |
| `Waiting` | Gas paused at its current radius |
| `Advancing` | Gas closing in |

Gas damage scales linearly with distance into the gas beyond 12 units (`unscaledDamageDist`), multiplied by `damageScaleFactor = 0.005`.

## Loot & Items

### Loot Radius (spawn drop radius by type)

| Item Type | Radius (units) |
|-----------|----------------|
| Gun | 3.4 |
| Ammo | 2.0 |
| Melee | 3.0 |
| Throwable | 3.0 |
| HealingItem | 2.5 |
| Armor | 3.0 |
| Backpack | 3.0 |
| Scope | 3.0 |
| Skin | 3.0 |
| Perk | 3.0 |

Loot has physics: `drag = 0.003` on normal terrain, `iceDrag = 0.0008` on ice.

### Weapon Speed Multipliers (base)

| Weapon Type | Speed Multiplier |
|-------------|-----------------|
| Gun | 0.88× |
| Melee | 1.00× |
| Throwable | 0.92× |

## Airdrop Model

| Constant | Value |
|----------|-------|
| Fall time | 8000 ms |
| Fly time | 30000 ms |
| Crush damage | 300 HP |

## Projectile Physics

| Parameter | Value |
|-----------|-------|
| Max height | 5 units |
| Gravity | 10 units/s² |
| Distance-to-mouse multiplier | 1.5× |
| Air drag | 0.7 |
| Ground drag | 3.0 |
| Ice drag | 1.0 |
| Water drag | 5.0 |

## Obstacle & Building Rotation

Obstacles support four rotation modes:

| Mode | Angles |
|------|--------|
| `Full` | Any angle (float) |
| `Limited` | 4 cardinal directions (0/90/180/270°) |
| `Binary` | 2 directions (0° or 90°) |
| `None` | Fixed, no rotation |

## Common Object Interface

All runtime objects implement a minimum shared interface (from `CommonObjectMapping`):

```typescript
{
    position: Vector
    rotation: number
    dead: boolean
    layer: Layer
    damageable?: boolean
    // plus category-specific fields
}
```

This is consumed by shared code in `common/` (bullets, collision detection, packet helpers).

## Documentation Index

See [content-plan.md](content-plan.md) for the full documentation index and status.

## Subsystem References

- [Definitions](subsystems/definitions/) — Full definition schemas for all types
- [Objects](subsystems/objects/) — Runtime object hierarchy and per-category implementation
- [Packets](subsystems/packets/) — How game state is serialized for network transport

## Related Documents

- **Tier 1:** [architecture.md](architecture.md) — System design and component map
- **Tier 1:** [protocol.md](protocol.md) — Binary protocol encoding for this data
- **Tier 1:** [development.md](development.md) — How to add new content

## Known Issues / Tech Debt

- **Definition index order:** Inserting a definition in the middle of a list shifts all subsequent indices and breaks protocol compatibility. Append-only when adding.
