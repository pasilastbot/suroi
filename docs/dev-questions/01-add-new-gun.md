# Q: How do I add a new gun to the game?

<!-- @tags: definitions, guns, protocol -->
<!-- @related: docs/subsystems/definitions/modules/items.md -->

## Overview

Adding a gun touches: definition (common), sprite assets (client), and protocol
version bump. Server logic and client rendering reuse existing infrastructure.

## 1. Add the gun definition

Open `common/src/definitions/items/guns.ts` and **append** to the definitions
array at the bottom. Copy an existing gun of similar type as a template.

Minimum required fields:

```typescript
{
    idString: "my_gun",
    name: "My Gun",
    defType: DefinitionType.Gun,

    ammoType: "9mm",          // references Ammos collection
    ammoSpawnAmount: 60,      // ammo dropped with gun on spawn
    capacity: 15,             // magazine size
    reloadTime: 2.0,          // seconds
    fireDelay: 90,            // ms between shots (auto/single)
    switchDelay: 300,         // ms after equip before can fire
    fireMode: FireMode.Single, // Single | Auto | Burst

    tier: Tier.B,             // S, A, B, C, D — affects airdrop quality

    damage: 20,               // damage per bullet
    ballistics: {
        damage: 20,
        obstacleMultiplier: 1,
        speed: 0.18,
        range: 100,
        tracer: { opacity: 0.8, width: 2, length: 1 }
    },

    recoilMultiplier: 0.8,    // speed multiplier while firing
    recoilDuration: 80,       // ms
    shotSpread: 2,            // degrees spread while still
    moveSpread: 3,            // additional spread while moving
    length: 7,                // barrel length (visual)

    speedMultiplier: 0.88,    // movement speed multiplier while equipped

    fists: {
        animationDuration: 100,
        left: Vec(38, -35),
        right: Vec(40, 0),
    },

    image: { position: Vec(60, 0) },
}
```

**Important:** Append to the end of the array. Inserting in the middle shifts
all subsequent indices and breaks protocol compatibility.

## 2. Add sprites

Create a loot SVG (used in inventory/ground):

```
client/public/img/game/shared/loot_<idString>.svg
```

And a world image SVG (held by the player):

```
client/public/img/game/shared/<idString>.svg
```

```bash
bun validateSvgs
```

## 3. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

Required because `Guns` is an `ObjectDefinitions` collection; adding a gun
changes serialization indices.

## 4. Validate

```bash
bun validateDefinitions
```

Fix any field validation errors before testing.

## 5. Add to a loot table (optional)

To make the gun spawn in the world, add it to a loot table in
`server/src/data/lootTables.ts`:

```typescript
{ item: "my_gun", weight: 1 }
```

See [Add Loot Table](11-add-loot-table.md).

## 6. Test

```bash
bun dev
```

Use the `give my_gun` dev command in the in-game console to test.

## Key Gun Fields Reference

| Field | Purpose |
|-------|---------|
| `fireMode` | `Single`, `Auto`, or `Burst` |
| `burstProperties` | `shotsPerBurst`, `burstCooldown` (only for Burst) |
| `bulletCount` | Pellets per shot (shotguns) |
| `reloadFullOnEmpty` + `fullReloadTime` | Different reload time when mag is empty |
| `spawnScope` | Scope that spawns with this gun |
| `noQuickswitch` | Prevents quickswitch abuse |
| `jitterRadius` | Random bullet position offset (shotguns) |
| `casingParticles` | Ejected shell casings |
| `dual` | Auto-generates a dual variant |

## Related

- [Items Module](../subsystems/definitions/modules/items.md) — file list
- [Bullets Module](../subsystems/definitions/modules/bullets.md) — bullet definitions auto-derived from ballistics
- [Protocol Version](04-protocol-version-bump.md) — when and how to bump
- [Definitions Patterns](../subsystems/definitions/patterns.md) — append-only rule
