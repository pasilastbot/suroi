# Map Generation Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/map.ts, server/src/data/maps.ts, server/src/data/lootTables.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Map Generation subsystem creates the procedural game world: terrain, rivers, obstacles, buildings, and loot. Each game uses a map definition (e.g. `normal`, `winter`) and a seed to produce a deterministic layout.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/map.ts` | `GameMap` — map generation, `generateObstacle`, `generateBuilding` |
| `server/src/data/maps.ts` | `Maps`, `MapDefinition`, `MapName` — map definitions |
| `server/src/data/lootTables.ts` | `LootTables` — loot table definitions by mode |
| `server/src/utils/lootHelpers.ts` | `getLootFromTable`, `getSpawnableLoots` — loot spawning |
| `common/src/utils/terrain.ts` | `Terrain`, `River`, `FloorTypes` — terrain types |

## Architecture

```
MapDefinition (maps.ts)
    ├── width, height, oceanSize, beachSize
    ├── rivers, trails — river definitions
    ├── clearings — open areas
    ├── buildings, majorBuildings — building counts
    ├── quadBuildingLimit — per-quadrant limits
    ├── obstacles — fixed obstacle counts
    └── bridges — bridge building types

GameMap (map.ts)
    ├── Constructor: generate terrain, rivers, buildings, obstacles
    ├── getRandomPosition() — spawn position finder
    ├── generateObstacle() — place obstacle with loot
    └── generateBuilding() — place building
```

## Data Flow

```
Game constructor
    → new GameMap(game, mapData, options)
    → Generate terrain (grass, water, beach)
    → Generate rivers (if defined)
    → Generate major buildings
    → Generate buildings (quad-based)
    → Generate obstacles (grass, river, etc.)
    → Generate clearings
    → Build MapPacket buffer (cached)

Obstacle spawn (loot table)
    → getLootFromTable(modeName, tableID);
    → resolveTable() → LootTables[mode][tableID]
    → weightedRandom() for items
    → Return LootItem[]
```

## Loot Tables

- **Format:** `LootTables[modeName][tableID]` — e.g. `LootTables.normal.ground_loot`
- **WeightedItem:** `{ item: idString, weight: number, count?: number }` or `{ table: string, weight: number }`
- **FullLootTable:** `{ min, max, noDuplicates?, loot }` for variable count
- **Nested tables:** `loot` can reference other tables by idString

## Map Definition Structure

| Field | Purpose |
|-------|---------|
| `width`, `height` | Map dimensions |
| `oceanSize`, `beachSize` | Ocean and beach margins |
| `rivers` | River definitions (width, obstacles) |
| `trails` | Trail definitions |
| `clearings` | Open areas with specific obstacles |
| `buildings` | Building idString → count |
| `majorBuildings` | Landmark buildings |
| `quadBuildingLimit` | Per-quadrant building limits |
| `obstacles` | Fixed obstacle counts (e.g. river chests) |

## Module Index (Tier 3)

- [Loot Tables](modules/loot-tables.md) — Loot spawning, weighted tables
- [Generation](modules/generation.md) — Terrain, rivers, buildings, obstacles flow
- [Terrain](modules/terrain.md) — FloorTypes, FloorNames, speed multiplier, slippery

## Protocol Considerations

- **Affects protocol:** Map data is in MapPacket. Terrain/obstacle format changes require protocol bump.

## Dependencies

- **Depends on:** Definitions (Obstacles, Buildings, Loots), LootHelpers
- **Depended on by:** Game (constructor), MapPacket

## Related Documents

- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — MapObjectSpawnMode, Layer
- **Tier 2:** [../definitions/](../definitions/) — Obstacle, Building definitions
- **Tier 2:** [../objects/](../objects/) — Obstacle, Building, Loot
