# Q: How does the bullet and ballistics system work end to end (fire, travel, hit detection)?

<!-- @tags: bullets, ballistics, server, hitbox, damage -->
<!-- @related: docs/subsystems/definitions/modules/bullets.md, docs/subsystems/objects/modules/server-objects.md -->

## Overview

```
Player fires
    → GunItem.useItem()
    → Game.addBullet()         — create Bullet server object
    → Bullet.update() per tick — move + hit detection
    → DamageRecord[] collected
    → Apply damage to hit objects
    → Client receives bullet data in UpdatePacket
    → Client renders tracer
```

## 1. Firing (Server, GunItem)

`server/src/inventory/gunItem.ts` — `useItem()`:

1. Compute bullet rotation = player aim direction + spread randomization
2. For each pellet (`bulletCount`): call `game.addBullet(idString, position, rotation, ...)`
3. The bullet `idString` is `{gunIdString}_bullet` (auto-derived from gun definition)
4. `game.addBullet()` creates a `Bullet` instance and adds it to `game.bullets`

## 2. Bullet Physics (Server, per tick step 7)

`server/src/objects/bullet.ts` — `update()`:

Each tick, every bullet:

1. Computes a line segment: `position → position + velocity × dt`
2. Constructs a `RectangleHitbox.fromLine()` for broad-phase collision
3. Queries the spatial grid: `grid.intersectsHitbox(lineRect, layer)`
4. For each intersecting object, calls `updateAndGetCollisions(dt, objects)`
   (from `BaseBullet` in `common/src/utils/baseBullet.ts`)
5. Collisions are checked per-object with fine hitbox intersection
6. If collision: record `DamageRecord {object, damage, weapon, source, position}`
7. Marks bullet `dead = true` if it hits a solid object or travels `maxDistance`

**Layer filtering:** Bullets only hit objects on adjacent or equal layers
(`adjacentOrEquivLayer`).

## 3. Damage Application (Server, tick step 8)

After all bullets update, `game.ts` processes collected `DamageRecord[]`:

```typescript
for (const record of damageRecords) {
    record.object.damage(record.damage, record.weapon, record.source);
}
```

`damage()` on `Obstacle`, `Building`, or `Player` reduces health, triggers
effects, calls `setDirty()`, and may trigger destruction or death.

## 4. Bullet Serialization (UpdatePacket)

Bullets are **client-side only** for rendering — the server does not serialize
`Bullet` objects as `ObjectCategory` instances. Instead, fired bullets are
included in the `UpdatePacket` as a compact array:

```
deserializedBullets[] in UpdatePacket
    → { idString, position, rotation, layer, ... }
```

The client creates a local `Bullet` object for tracer rendering. The client
does not simulate damage — all damage is authoritative on the server.

## 5. Client Rendering (ClientBullet)

`client/src/scripts/objects/bullet.ts` renders the tracer line:

- `tracer.opacity`, `tracer.width`, `tracer.length` from `BaseBulletDefinition`
- Colors derived from ammo type or explicit `tracer.color`
- `trail` field enables particle trails
- Bullets move client-side each frame for smooth rendering (not synced after
  initial spawn)

## BaseBulletDefinition Fields That Affect Ballistics

| Field | Effect |
|-------|--------|
| `speed` | Units per ms |
| `range` | Max distance before disappearing |
| `rangeVariance` | Random ± range variation per shot |
| `damage` | Damage per hit |
| `obstacleMultiplier` | Damage × this vs obstacles |
| `shrapnel` | Shrapnel coloring |
| `noReflect` | Cannot bounce off reflective surfaces |
| `onHitExplosion` | Triggers this explosion on impact |
| `explodeOnImpact` | Explodes immediately on hit vs penetrate |
| `allowRangeOverride` | Can be thrown further (throwable guns) |

## Damage Modifiers (Runtime)

Damage applied to a target accounts for:

- `modifiers.damage` — shooter's active WearerAttributes fire rate/damage modifier
- `modifiers.dtc` (damage to cover) — obstacle damage multiplier
- `definition.obstacleMultiplier` — from bullet definition
- `reflectionCount` — damage reduces by `1 / (reflectionCount + 1)` per bounce

## Related

- [Bullets Module](../subsystems/definitions/modules/bullets.md) — BaseBulletDefinition reference
- [Server Objects](../subsystems/objects/modules/server-objects.md) — Bullet server object
- [Tick Serialization](../subsystems/game-loop/modules/tick-serialization.md) — tick step 7-8 (bullets, damage)
- [WearerAttributes](30-wearer-attributes.md) — how perks modify bullet damage
