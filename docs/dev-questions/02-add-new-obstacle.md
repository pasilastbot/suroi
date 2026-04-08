# Q: How do I add a new obstacle to the game?

<!-- @tags: definitions, obstacles, loot, SVG -->
<!-- @related: docs/subsystems/definitions/modules/obstacles.md -->

## 1. Add the obstacle definition

Open `common/src/definitions/obstacles.ts` and append to the array:

```typescript
{
    idString: "my_obstacle",
    name: "My Obstacle",
    defType: DefinitionType.Obstacle,

    health: 100,
    scale: {
        spawnMin: 1.0,
        spawnMax: 1.0,
        destroy: 0.5,  // scale when destroyed (for rubble effect)
    },

    hitbox: new CircleHitbox(2.5),  // or RectangleHitbox, PolygonHitbox, GroupHitbox

    // Rotation
    rotationMode: RotationMode.Limited,  // Full | Limited | Binary | None

    // Material for sounds and particle effects
    material: "wood",  // wood, stone, metal, bush, glass, etc.

    // Loot
    lootTable: "my_obstacle_loot",  // references LootTables (optional)
    spawnWithLoot: true,            // spawn with loot pre-placed inside

    // Bullet behavior
    allowFlyover: FlyoverPref.Always,  // Always | Sometimes | Never
    // impenetrable: true,            // bullets cannot pass through
    // noBulletCollision: true,       // bullets pass through entirely
    // reflectBullets: true,          // bullets bounce off

    // Destruction
    // explosion: "barrel_explosion", // explosion on destroy (optional)

    // Appearance
    // zIndex: ZIndexes.ObstaclesLayer2,  // default
    // tint: 0xffffff,  // optional color tint
    // variations: 4,   // how many sprite variants (my_obstacle_1, my_obstacle_2, ...)
}
```

## 2. Add sprites

**Active state** (normal):

```
client/public/img/game/shared/obstacles/my_obstacle.svg
```

**Destroyed state** (optional rubble):

```
client/public/img/game/shared/obstacles/my_obstacle_residue.svg
```

If using `variations: N`, name them `my_obstacle_1.svg`, `my_obstacle_2.svg`, etc.

```bash
bun validateSvgs
```

## 3. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

## 4. Add to a loot table (if applicable)

In `server/src/data/lootTables.ts`:

```typescript
LootTables["normal"]["my_obstacle_loot"] = [
    { item: "ak47", weight: 1 },
    { item: "556mm", weight: 3 },
];
```

## 5. Validate

```bash
bun validateDefinitions
bun validateSvgs
```

## 6. Add to a map

To make it spawn in the world, reference it in a `MapDefinition` in
`server/src/data/maps.ts`:

```typescript
obstacles: {
    my_obstacle: 20,  // spawn 20 of these
}
```

Or add it to clearings/river banks via the map's `clearings` or `rivers` config.

## 7. Test

```bash
bun dev
```

## Key Fields Reference

| Field | Purpose |
|-------|---------|
| `health` | HP before destruction |
| `hitbox` | Collision shape (`CircleHitbox`, `RectangleHitbox`, `PolygonHitbox`, `GroupHitbox`) |
| `material` | Sound effects category |
| `hardness` | Melee weapons with `maxHardness >= this` can destroy it |
| `rotationMode` | How obstacle is rotated in world |
| `allowFlyover` | Whether throwables can arc over |
| `variations` | Number of sprite variants |
| `invisible` | No sprite (invisible collision only) |
| `isWall` | Used for building wall obstacles |
| `isDoor` | Enables door open/close interaction |
| `interactObstacleIdString` | Door toggles this obstacle's state |

## Related

- [Obstacles Module](../subsystems/definitions/modules/obstacles.md) — full field reference
- [Loot Tables](11-add-loot-table.md) — how to configure what drops
- [Map Generation](../subsystems/map/modules/generation.md) — adding obstacles to maps
