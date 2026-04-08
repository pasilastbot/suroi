# Q: How do I add or modify a map?

<!-- @tags: map, generation, terrain, buildings -->
<!-- @related: docs/subsystems/map/modules/generation.md, docs/subsystems/map/README.md -->

## 1. Add the map definition

Open `server/src/data/maps.ts` and add a new entry to the `maps` object:

```typescript
const maps = {
    // ...existing maps...
    my_map: {
        width: 1632,
        height: 1632,
        oceanSize: 128,      // ocean border width
        beachSize: 32,       // beach between ocean and land

        // Terrain
        rivers: {
            minAmount: 1,
            maxAmount: 2,
            maxWideAmount: 1,
            wideChance: 0.35,
            minWidth: 12,
            maxWidth: 18,
            minWideWidth: 25,
            maxWideWidth: 30,
            obstacles: {
                river_rock: 16,
            }
        },

        // Buildings
        majorBuildings: ["headquarters"],  // landmark buildings (placed at map center)
        buildings: {
            warehouse: 3,
            porta_potty: 12,
            container_3: 20,
        },
        quadBuildingLimit: {
            warehouse: 1,   // max 1 warehouse per map quadrant
        },

        // Obstacles
        obstacles: {
            oak_tree: 50,
            bush: 80,
            stone_wall: 40,
        },

        // Optional: Ground loot spawners
        loots: {
            ground_loot: 50,
        },
    },
} satisfies Record<string, MapDefinition>;
```

## 2. Export and register the map name

In `server/src/data/maps.ts`, the `Maps` export and `MapName` type are auto-derived
from the `maps` object. Adding a key to `maps` automatically registers it.

## 3. Add a spritesheet variant (optional)

If your map has custom visual assets (winter trees, desert obstacles):

1. Create a new variant directory:
   ```
   client/public/img/game/<variant>/
   ```
2. Add SVG assets named to override the shared sprites
3. Add the mode's `spriteSheets` reference (see [Add Game Mode](06-add-new-game-mode.md))

## 4. Configure the server to use your map

Edit `server/config.json`:

```json
{
    "map": "my_map"
}
```

Or use the `map` config option in your test environment.

## 5. Test

```bash
bun dev
```

Join a game — the new map layout should generate. Each run with a new seed
produces a different layout.

## Modifying an Existing Map

Edit the map definition in `server/src/data/maps.ts`. Changes take effect
on server restart.

**No protocol version bump required** for map changes — maps are generated
server-side and sent via `MapPacket`. Only adding new obstacle/building
*definitions* (not uses) requires a protocol bump.

## MapDefinition Field Reference

| Field | Purpose |
|-------|---------|
| `width`, `height` | Map dimensions in world units |
| `oceanSize` | Ocean border width |
| `beachSize` | Beach strip between ocean and land |
| `rivers` | River generation parameters |
| `trails` | Trail generation (similar to rivers) |
| `clearings` | Open areas with specific obstacle sets |
| `majorBuildings` | Landmark buildings (placed first, centered) |
| `buildings` | `{ idString: count }` for regular buildings |
| `quadBuildingLimit` | Max count per quadrant for specified buildings |
| `obstacles` | `{ idString: count }` for standalone obstacles |
| `loots` | `{ tableId: count }` for ground loot spawners |
| `places` | Named location labels shown on minimap |
| `onGenerate` | Custom generation callback (advanced) |

## Related

- [Map Generation](../subsystems/map/modules/generation.md) — full generation pipeline
- [Map Subsystem](../subsystems/map/README.md) — overview
- [Loot Tables](11-add-loot-table.md) — configuring what spawns
- [Add Building](03-add-new-building.md) — creating buildings for the map
