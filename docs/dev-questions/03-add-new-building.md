# Q: How do I add a new building to the game?

<!-- @tags: definitions, buildings, map, layers -->
<!-- @related: docs/subsystems/definitions/modules/buildings.md -->

## Overview

Buildings are the most complex definition type. A building is composed of:
- **Floor images** — rendered below players (ground texture)
- **Ceiling images** — rendered above players (roof)
- **Wall/hitbox** — collision geometry
- **Obstacles** — doors, crates, etc. placed inside
- **Sub-buildings** — nested buildings (basement, separate rooms)
- **Loot spawners** — fixed loot spawn points

## 1. Plan your building

Before coding, decide:
- How many floors? (1 = simple, 2+ = layers needed)
- Does it have doors?
- What obstacle layout?
- What SVG assets are needed?

## 2. Add the building definition

Open `common/src/definitions/buildings.ts` and append:

```typescript
{
    idString: "my_building",
    name: "My Building",
    defType: DefinitionType.Building,

    // Spawn hitbox — area that must be clear for placement
    spawnHitbox: new RectangleHitbox(Vec(-20, -15), Vec(20, 15)),

    // Collision hitbox (optional) — walls players bump into
    hitbox: new GroupHitbox(
        new RectangleHitbox(Vec(-20, -15), Vec(-18, 15)),  // left wall
        new RectangleHitbox(Vec(18, -15), Vec(20, 15)),   // right wall
        new RectangleHitbox(Vec(-20, -15), Vec(20, -13)), // top wall
        new RectangleHitbox(Vec(-20, 13), Vec(20, 15)),   // bottom wall
    ),

    // Floor images
    floorImages: [
        {
            key: "my_building_floor",
            position: Vec(0, 0),
        }
    ],

    // Ceiling images
    ceilingImages: [
        {
            key: "my_building_ceiling",
            position: Vec(0, 0),
        }
    ],

    // Ceiling scope — what scope zoom inside the building
    // ceilingScope: "2x",  // optional

    // Obstacles inside the building
    obstacles: [
        {
            idString: "regular_crate",
            position: Vec(-10, 5),
        },
        {
            idString: "door",
            position: Vec(0, 15),  // bottom door
            rotation: 0,
        },
    ],

    // Fixed loot spawners (optional)
    lootSpawners: [
        {
            position: Vec(5, -5),
            table: "ground_loot",
        }
    ],

    // Floor type (affects movement speed)
    floors: [
        {
            type: FloorNames.Wood,
            hitbox: new RectangleHitbox(Vec(-18, -13), Vec(18, 13)),
        }
    ],
}
```

## 3. Add floor sprites

The `key` in `floorImages` and `ceilingImages` maps to a sprite frame name.

```
client/public/img/game/shared/<key>.svg
```

```bash
bun validateSvgs
```

## 4. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

## 5. Validate

```bash
bun validateDefinitions
```

## 6. Add to a map

In `server/src/data/maps.ts`:

```typescript
buildings: {
    my_building: 3,
}
```

## 7. Test

```bash
bun dev
```

## Multi-Floor Buildings (Layers)

For a building with a basement or upstairs, use `layer` offsets on obstacles
and sub-buildings:

```typescript
// Basement
subBuildings: [
    {
        idString: "my_basement",
        position: Vec(0, 0),
        layer: Layer.Basement,  // offset from parent layer
    }
],
// Staircase area
obstacles: [
    {
        idString: "staircase",
        position: Vec(5, 0),
        layer: Layer.ToBasement,  // transition zone
    }
]
```

Players in a `ToBasement` / `ToUpstairs` layer zone can interact with objects
on both adjacent layers.

See [Layer System](21-layer-system.md) for a full explanation.

## Key BuildingDefinition Fields

| Field | Purpose |
|-------|---------|
| `spawnHitbox` | Area that must be clear during map generation |
| `hitbox` | Wall/collision hitbox (can be `GroupHitbox` for complex shapes) |
| `ceilingHitbox` | Hitbox for ceiling collapse (if different from `hitbox`) |
| `floorImages` | Ground texture images (rendered below players) |
| `ceilingImages` | Roof images (rendered above players) |
| `floors` | Floor type regions (affects movement speed) |
| `obstacles` | Obstacles placed inside |
| `subBuildings` | Nested buildings |
| `lootSpawners` | Fixed loot spawn points |
| `ceilingScope` | Scope zoom when inside |
| `collideWithLayers` | Which layers this building's hitbox affects |
| `hideOnMap` | Don't show on minimap |
| `orientation` | Whether building can be rotated (0 = no, 1 = binary) |

## Related

- [Buildings Module](../subsystems/definitions/modules/buildings.md) — field reference
- [Layer System](21-layer-system.md) — multi-floor architecture
- [Add Map](10-add-new-map.md) — placing the building in the world
- [Add Obstacle](02-add-new-obstacle.md) — obstacles placed inside buildings
