# Loot & Pickup — Loot Generation & Spawn Tables

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/map/README.md -->
<!-- @source: server/src/data/lootTables.ts, server/src/utils/lootHelpers.ts, server/src/map.ts, common/src/definitions/loots.ts -->

## Purpose
Defines loot table generation system: per-location loot tables, rarity weights, recursive table resolution, and spawn density control. Ensures balanced item distribution across the island.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/data/lootTables.ts` | LootTable definitions per mode and location | High |
| `server/src/utils/lootHelpers.ts` | LootTable resolution, weighted random selection, spawn helpers | High |
| `server/src/map.ts` | Obstacle/building loot integration, spawn point generation | High |
| `common/src/definitions/loots.ts` | LootDefinition registry, item type validation | Medium |

## Business Rules

- **Loot Table Structure:** Nested tree of tables → items with weights (probabilities)
- **Mode-Specific Tables:** "normal" mode, "winter" mode each have separate tables (some shared)
- **Location-Based:** Buildings/obstacles have specific `lootTable` property referencing table by ID
- **Weighted Selection:** Items picked randomly per weight; higher weight = more likely
- **Recursive Resolution:** Tables can reference other tables (e.g., "chest_rare" → "high_tier_guns")
- **Quality Filters:** Optional `quality` parameter filters items by tier/rarity within a table
- **Table Caching:** Resolved tables cached to avoid re-parsing
- **Spawn Density:** Config controls density multiplier across entire island

## Data Lineage

### Loot Table Hierarchy
```
LootTables definition (server/data/lootTables.ts)
  ├─ normal_tables
  │   ├─ "building_house": [
  │   │     { item: "common_ammo", weight: 10 },
  │   │     { item: "gauze", weight: 5 },
  │   │     { table: "house_special", weight: 1 }  ← nested table reference
  │   │   ]
  │   └─ "house_special": [
  │       { item: "ar_15", weight: 3 },
  │       { item: "m16a4", weight: 2 }
  │     ]
  └─ winter_tables
      ├─ "frozen_building": [...]
      └─ ...
```

### Loot Spawn Flow
```
Map generation: Building placed at position
  ↓
Building has lootTable: "building_house"
  ↓
Obstacle/chest created
  ↓
Game.map.generateObstacle(buildingDef, position)
  ↓
Obstacle.onPlaced() called
  ↓
getLootFromTable(modeName, "building_house")
  ↓
LootHelpers.resolveTable("building_house", modeName)
  ├─ Check cache for resolved table
  ├─ If miss: parse table, follow nested references
  └─ Return resolved item list with weights
  ↓
For each spawn point in obstacle:
    Pick random item from list by weight
    ↓
    Loot object created with:
        definition: item definition
        count: quantity (1 for most, varies for ammo)
        position: spawn point
  ↓
Loot added to game.loots[] and spatial grid
```

### Weighted Selection
```
Table resolved to list:
[
  { item: "common_ammo", weight: 10 },
  { item: "gauze", weight: 5 },
  { item: "ar_15", weight: 3 }
]

Total weight = 10 + 5 + 3 = 18
  ↓
Random number 0–18 selected
  ↓
Random = 7:  falls in "common_ammo" range (0–10) → pick ammo
Random = 12: falls in "gauze" range (10–15) → pick gauze
Random = 17: falls in "ar_15" range (15–18) → pick AR-15
```

## Complex Functions

### `getLootFromTable(modeName, tableID, quality?)` — @file server/src/utils/lootHelpers.ts
**Purpose:** Get a single loot item from a table using weighted random selection.

**Implicit behavior:**
- Calls `resolveTable(tableID, modeName)` to get unexpanded table (may have nested references)
- Recursively expands the table:
  - For each `{ item: ... }` entry: adds to item list
  - For each `{ table: ... }` entry: recursively expands that table
  - Flattens results into single list
- Filters by quality if provided (compares item tier against quality threshold)
- Performs weighted random selection: `weightedRandom(expandedItems)`
- Returns the selected item definition or throws error if table missing

**Called by:** Obstacle/building loot spawn, enemy drops, airdrop contents

**Example:**
```typescript
// Get random item from "chest_rare" table in normal mode
const loot = getLootFromTable("normal", "chest_rare");
// May expand as:
// chest_rare → [ak47 (w=5), m16a4 (w=3), uzi (w=2)]
// Weighted selection returns one of those
```

### `resolveTable(tableID, modeName)` — @file server/src/utils/lootHelpers.ts
**Purpose:** Load loot table by ID and mode, with caching.

**Implicit behavior:**
- Checks cache first: if `LootTableCache[tableID]` exists, return immediately
- If cache miss:
  - Look up tableID in LootTables[modeName][...][tableID]
  - If not found in mode-specific: fallback to LootTables.normal
  - If still not found: throw ReferenceError
  - Cache the resolved table
- Returns the table array (may still contain nested references)

**Called by:** `getLootFromTable()`

**Side effects:**
- Populates LootTableCache (one-time per unique table)
- Lazy caching: table only cached when first accessed

## Loot Table Format

```typescript
interface LootTable {
    readonly [name: string]: Array<
        | { readonly item: string; readonly weight: number }
        | { readonly table: string; readonly weight: number }
        | { readonly item?: string; readonly table?: string; readonly weight: number; readonly quality?: number }
    >
}
```

### Example Table Definition
```typescript
// In lootTables.ts
gun_crate: [
    { item: "ak47", weight: 5 },            // 5/16 chance
    { item: "m16a4", weight: 4 },           // 4/16 chance
    { item: "scout", weight: 3 },           // 3/16 chance
    { table: "attachments_common", weight: 4 }  // Nested table
]

attachments_common: [
    { item: "holosight", weight: 3 },
    { item: "ak_muzzle", weight: 1 }
]
```

## Configuration

| Setting | Source | Effect | Default |
|---------|--------|--------|---------|
| `modeName` | Enum | "normal" or "winter" | determined by game mode |
| `quality` | Parameter | Filter items ≥ quality tier (optional) | undefined (no filter) |
| `spawnDensity` | Config | Global multiplier on loot spawn count | 1.0 |
| `Config.loot.disableLootCache` | config.json | Force table re-parse each spawn (slow, debug only) | false |

## Loot Distribution Examples

### Gun Crate (High-Tier)
Table: "gun_crate_rare"
- **AR**s: 40%
- **Scout Rifle**: 30%
- **Sniper**: 20%
- **Shotgun**: 10%

### Small Building (Low-Tier)
Table: "building_house"
- **Ammo**: 50%
- **Healing**: 30%
- **Common Gun**: 15%
- **Rare Gun**: 5%

### Winter Mode (Variant)
Table: "frozen_crate"
- Overlaps with normal loot tables
- May have winter-exclusive items (e.g., frost rounds)

## Known Gotchas

- **Circular References:** If Table A → Table B → Table A, causes infinite loop. Validation should catch this.
- **Missing Table:** Referencing undefined table throws ReferenceError; game crashable in production.
- **Cache Stale:** Changing LootTables.ts requires server restart; no live reload mechanism.
- **Quality Mismatch:** Quality filter may match no items, returning undefined (should fall back to unfiltered).

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Map generation, obstacle placement, loot integration
- **Tier 2:** [../../inventory/README.md](../../inventory/README.md) — Loot pickup, item management
- **Tier 2:** [../../server-utilities/README.md](../../server-utilities/README.md) — Weighted random helpers
- **Tier 1:** [../../../../development.md](../../../../development.md) — Testing loot tables, config
- **Tier 1:** [../../../../datamodel.md](../../../../datamodel.md) — LootTable structure
