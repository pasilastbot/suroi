# Map — Generation Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/map/README.md -->
<!-- @source: server/src/map.ts, server/src/data/maps.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the procedural map generation flow: terrain, rivers, buildings, obstacles, and clearings. Each game uses a map definition and seed for deterministic layout.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/map.ts | `GameMap` — constructor runs full generation pipeline | High |
| @file server/src/data/maps.ts | `Maps`, `MapDefinition`, `MapName` — map configs | Medium |
| @file common/src/utils/terrain.ts | `Terrain`, `River`, `FloorTypes` | Medium |

## Generation Order (constructor)

1. **Terrain** — Grass, water, beach from `oceanSize`, `beachSize`
2. **Rivers** — Generate rivers from `rivers` definitions
3. **Major buildings** — Landmark buildings, quad-based placement
4. **Buildings** — Regular buildings, `quadBuildingLimit` per quadrant
5. **Obstacles** — Grass, river, clearing obstacles
6. **Clearings** — Open areas with specific obstacle sets
7. **MapPacket buffer** — Serialize and cache for all players

## MapDefinition Fields

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

## Data Lineage

```
MapDefinition → GameMap constructor
    → terrain (FloorTypes)
    → rivers (River, bridges)
    → buildings (Building, SubBuilding)
    → obstacles (Obstacle + loot from loot tables)
    → MapPacket buffer
```

## Business Rules

- Same seed + mapDef produces identical layout
- Quad-based building limits prevent clustering
- Obstacles use `getLootFromTable()` for loot spawn

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Map overview
- **Tier 3:** [../modules/loot-tables.md](loot-tables.md) — Loot spawning
- **Tier 2:** [../definitions/](../../definitions/) — Obstacle, Building definitions
