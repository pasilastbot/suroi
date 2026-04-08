# Map — Loot Tables Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/map/README.md -->
<!-- @source: server/src/data/lootTables.ts, server/src/utils/lootHelpers.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

Loot tables define what items spawn from obstacles, crates, and other sources. Each table is keyed by mode (e.g. `normal`, `halloween`) and table ID (e.g. `ground_loot`, `barrel`).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/data/lootTables.ts` | `LootTables` — all table definitions | High |
| `server/src/utils/lootHelpers.ts` | `getLootFromTable`, `getLoot`, `resolveTable` | Medium |

## Business Rules

- **WeightedItem:** `{ item: idString, weight: number }` or `{ table: string, weight: number }` for nested tables
- **FullLootTable:** `{ min, max, noDuplicates?, loot }` — spawn random count between min–max
- **SimpleLootTable:** Array of WeightedItem or array of arrays (one pick per inner array)
- **quality:** Optional parameter for tier-based filtering (e.g. airdrop quality)

## Data Flow

```
Obstacle destroyed / crate opened
    → getLootFromTable(modeName, tableID, quality?)
    → resolveTable() → LootTables[mode][tableID]
    → getLoot() — weighted random selection
    → Return LootItem[] (idString + count)
    → Game.addLoot() for each
```

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Map overview
- **Tier 2:** [../../definitions/](../../definitions/) — Loot definitions
