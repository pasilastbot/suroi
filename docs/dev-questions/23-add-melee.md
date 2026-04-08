# Q: How do I add a new melee weapon?

<!-- @tags: definitions, melees, inventory, hitbox -->
<!-- @related: docs/subsystems/definitions/modules/items.md, docs/subsystems/inventory/modules/weapons.md -->

## 1. Add the melee definition

Open `common/src/definitions/items/melees.ts` and append:

```typescript
{
    idString: "my_melee",
    name: "My Melee",
    defType: DefinitionType.Melee,

    tier: Tier.B,             // S, A, B, C, D
    damage: 45,
    obstacleMultiplier: 1.5,  // damage × this vs obstacles

    // Optional: only damage obstacles up to this hardness level
    // maxHardness: 2,

    // Hit geometry
    radius: 2.0,              // hit radius from player center
    offset: Vec(3, 0),        // hit circle center offset (forward)

    // Timing
    cooldown: 400,            // ms between swings
    hitDelay: 150,            // ms after swing starts before hit registers

    speedMultiplier: 1.0,     // player speed while equipped

    // Sounds
    swingSound: "swing",
    hitSound: "soft_hit",

    // Optional visual
    image: {
        position: Vec(42, 20),
        angle: 135,
        lootScale: 0.6,
    },

    fists: {
        animationDuration: 150,
        left: Vec(40, -25),
        right: Vec(40, 15),
    },

    // Swing animation keyframes
    animation: [
        {
            duration: 100,
            fists: {
                left: Vec(40, 25),
                right: Vec(0, 50),
            },
        },
        {
            duration: 150,
            fists: {
                left: Vec(0, -50),
                right: Vec(40, -25),
            },
        },
        {
            duration: 150,
            fists: {
                left: Vec(40, -25),
                right: Vec(40, 15),
            },
        },
    ],
}
```

## 2. Add sprites

Loot sprite (inventory/ground):
```
client/public/img/game/shared/loot_<idString>.svg
```

World sprite (held by player, optional):
```
client/public/img/game/shared/<idString>.svg
```

If `image` is not in the definition, the player uses default fist animations.

```bash
bun validateSvgs
```

## 3. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

## 4. Validate

```bash
bun validateDefinitions
```

## 5. Add to a loot table

```typescript
// server/src/data/lootTables.ts
{ item: "my_melee", weight: 1 }
```

## 6. Test

```bash
bun dev
```

Use `give my_melee` in the dev console.

## How Melee Attacks Work at Runtime

1. Player presses attack → `MeleeItem.useItem()` called
2. After `hitDelay` ms → `getMeleeHitbox()` computes hit circle position
3. `getMeleeTargets()` queries the grid for objects in range
4. Each target within `radius` of offset receives `damage` (× `obstacleMultiplier` for obstacles)
5. `cooldown` timer prevents immediate re-swing

## Key Fields Reference

| Field | Purpose |
|-------|---------|
| `maxHardness` | Only damages obstacles with `hardness ≤ maxHardness` |
| `piercingMultiplier` | Damage multiplier vs pierceable objects |
| `stonePiercing` | Can damage stone objects |
| `fireMode` | `Single` (one swing) or `Auto` (hold to swing) |
| `maxTargets` | Max number of targets hit per swing |
| `numberOfHits` | Multi-hit per swing (advanced) |
| `reflectiveSurface` | Bullets from reflective weapons (mirrors) |
| `onBack` | Alternate position when on player's back |

## Related

- [Items Module](../subsystems/definitions/modules/items.md) — melees.ts
- [Weapons Module](../subsystems/inventory/modules/weapons.md) — MeleeItem runtime
- [Protocol Version](04-protocol-version-bump.md) — when and how to bump
