# Q: How do I add a new throwable weapon (grenade, C4, etc.)?

<!-- @tags: definitions, throwables, explosions, inventory -->
<!-- @related: docs/subsystems/definitions/modules/items.md, docs/subsystems/definitions/modules/explosions.md -->

## Overview

A throwable needs:
1. A `ThrowableDefinition` in `throwables.ts`
2. An `ExplosionDefinition` it references (can be new or existing)
3. SVG assets
4. Protocol version bump

## 1. Add (or reuse) an explosion

If your throwable has a custom explosion, first add it (see [Add Explosion](22-add-explosion.md)):

```typescript
// common/src/definitions/explosions.ts
{
    idString: "my_throwable_explosion",
    damage: 80,
    obstacleMultiplier: 1.5,
    radius: { min: 4, max: 10 },
    animation: { duration: 800, tint: 0xff8800, scale: { start: 1, end: 2 } },
    cameraShake: { duration: 200, intensity: 4 },
    shrapnelCount: 8,
    ballistics: {
        damage: 12,
        obstacleMultiplier: 1,
        speed: 0.15,
        range: 18,
        shrapnel: true,
    },
    decal: "explosion_decal",
    decalFadeTime: 15000,
}
```

## 2. Add the throwable definition

Open `common/src/definitions/items/throwables.ts` and append:

```typescript
{
    idString: "my_grenade",
    name: "My Grenade",
    defType: DefinitionType.Throwable,
    itemType: ItemType.Throwable,

    ammoSpawnAmount: 2,     // how many spawn with player
    speedMultiplier: 0.92,  // player speed while holding

    // Throw physics
    cookTime: 1500,         // ms to hold before throwing (C4-style: use 0)
    throwTime: 150,         // ms of throw animation
    fuseTime: 4000,         // ms from throw to explosion (set to 0 for instant)

    // Or for a cook-and-throw grenade (explodes when fuseTime reached):
    // cookTime: 0,
    // fuseTime: 4000,      // full fuse from throw

    explosionType: "my_throwable_explosion",

    // World model physics
    radius: 0.8,
    speed: 1.5,             // throw speed
    maxThrowDistance: 60,   // max throw range
    hitSomethingSound: "stone_hit",

    // Appearance
    image: {
        position: Vec(60, 10),
        angle: 0,
        lootScale: 0.7,
    },
    fists: {
        animationDuration: 150,
        left: Vec(38, -35),
        right: Vec(40, 0),
    },
}
```

## 3. Add sprites

Loot sprite (inventory/ground):
```
client/public/img/game/shared/loot_<idString>.svg
```

World sprite (when thrown, in-flight):
```
client/public/img/game/shared/<idString>.svg
```

```bash
bun validateSvgs
```

## 4. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

Both `Throwables` and `Explosions` definition lists change.

## 5. Validate

```bash
bun validateDefinitions
```

## 6. Add to a loot table

```typescript
// server/src/data/lootTables.ts
LootTables["normal"]["ground_loot"] = [
    // ...
    { item: "my_grenade", weight: 1 },
];
```

## 7. Test

```bash
bun dev
```

Use `give my_grenade` in the dev console to test.

## How Throwables Work at Runtime

1. Player equips throwable slot → `ThrowableItem` is the active item
2. Player holds attack → `cookTime` timer starts (if > 0)
3. Player releases → `ThrowableItem.throw()` creates a `Projectile` server object
4. `Projectile` arcs with gravity (see `datamodel.md` projectile physics)
5. When `fuseTime` expires or projectile hits something → `Game.addExplosion()`
6. Explosion processes damage, shrapnel, decal

For C4-style (placed then detonated): use the `ExplodeC4` `InputAction` after placing.

## Related

- [Items Module](../subsystems/definitions/modules/items.md) — throwables.ts
- [Explosions Module](../subsystems/definitions/modules/explosions.md) — explosion field reference
- [Add Explosion](22-add-explosion.md) — creating the explosion definition
- [Weapons Module](../subsystems/inventory/modules/weapons.md) — ThrowableItem runtime
