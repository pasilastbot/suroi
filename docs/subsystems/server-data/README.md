# Server Data Systems

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/data/, common/src/defaultInventory.ts -->

## Purpose

Static game configuration data loaded at server startup: gas zone progression, loot spawn probability tables, playable maps, and default player inventory. This subsystem provides 14-stage gas progression (0–6 min 30 sec), weighted loot distributions by container type, and 11 playable/debug map presets with obstacle/building layouts.

## Key Files & Entry Points

| System | File | Purpose |
|--------|------|---------|
| **Gas Progression** | [server/src/data/gasStages.ts](../../../../server/src/data/gasStages.ts) | 14-stage gas zone shrinkage with radius % and damage/tick |
| **Loot Distribution** | [server/src/data/lootTables.ts](../../../../server/src/data/lootTables.ts) | Weighted random item tables by container type + mode |
| **Map Definitions** | [server/src/data/maps.ts](../../../../server/src/data/maps.ts) | 11 maps with spawn/buildings/obstacles/biomes |
| **Default Inventory** | [common/src/defaultInventory.ts](../../../../common/src/defaultInventory.ts) | Starting item counts (generated per definition type) |

## Gas Stages Data

### Overview

14 sequential stages span **6 minutes 30 seconds** (390 total seconds). Each stage defines:
- **State** — `Inactive` → `Waiting` → `Advancing` → final `Waiting`
- **Radius** — % of map width from center (7 radius tiers: 0.76 → 0.55 → 0.43 → 0.32 → 0.2 → 0.09 → 0)
- **Damage/tick (DPS)** — health depletion outside safe zone
- **Airdrops** — summoned at stages 3, 7, 11 (`summonAirdrop: true`)
- **Final Stage** — stages 12–13 are marked `finalStage: true` (client UI only)

### Stage Progression Table

| Stage | State | Duration (s) | From Radius | To Radius | DPS | Airdrop | Cumulative Time |
|-------|-------|-------------|------------|-----------|-----|---------|------------------|
| 0 | Inactive | 0 | 0.76 | 0.76 | 0 | — | 0 s |
| 1 | Waiting | 75 | 0.76 | 0.55 | 0 | — | 75 s |
| 2 | Advancing | 20 | 0.76 | 0.55 | 1 | — | 95 s |
| 3 | Waiting | 45 | 0.55 | 0.43 | 1 | ✓ | 140 s (2:20) |
| 4 | Advancing | 20 | 0.55 | 0.43 | 1 | — | 160 s |
| 5 | Waiting | 40 | 0.43 | 0.32 | 2 | — | 200 s (3:20) |
| 6 | Advancing | 15 | 0.43 | 0.32 | 2 | — | 215 s |
| 7 | Waiting | 35 | 0.32 | 0.2 | 3 | ✓ | 250 s (4:10) |
| 8 | Advancing | 15 | 0.32 | 0.2 | 3 | — | 265 s |
| 9 | Waiting | 30 | 0.2 | 0.09 | 5 | — | 295 s (4:55) |
| 10 | Advancing | 15 | 0.2 | 0.09 | 5 | — | 310 s |
| 11 | Waiting | 20 | 0.09 | 0 | 10 | ✓ | 330 s (5:30) |
| 12 | Advancing | 60 | 0.09 | 0 | 10 | — | 390 s (6:30) |
| 13 | Waiting | 0 | 0 | 0 | 10 | — | 390 s (final) |

### Safety & Damage

- **GAME_SPAWN_WINDOW = 84 seconds** — players cannot join after 1:24. Soft close at stage 3 (2:20), hard close at stage 4.
- **Stage 13 damage unbounded** — final stage damage (10 DPS) with no time limit = indefinite damage. Caught in final zone = guaranteed death (design intent: forces race to center).

## Loot Tables

### Organization

Loot tables grouped by **mode** + **container type**. Each container type has a weighted list of item references or sub-table references.

#### Mode-Based Tables (server/src/data/lootTables.ts)

Defined for modes: `normal`, `fall`, `halloween`, `infection`, `hunted`, `winter`

#### Container Types (Common)

- `ground_loot` — scattered items around map
- `regular_crate` — common wooden crates (100 per normal map)
- `hazel_crate` — special event crates
- `viking_chest` — themed chest with strong guns (1 per map, 5 items)
- `river_chest` — river hazard chest (1 per map, 5 items)
- `frozen_crate` — winter mode special crate
- `aegis_crate`, `flint_crate`, `nsd_crate`, `lansirama_crate` — special high-tier crates (3–5 items each)
- `dumpster` — scrap items (2–4 items)
- `grenade_crate`, `grenade_box` — throwables (1–3 items)
- `melee_crate` — melee weapons (2 items)

### Entry Structure

```typescript
interface LootEntry {
  item?: string;        // Direct item idString (e.g., "ak47", "bandage")
  table?: string;       // Reference to sub-table (e.g., "guns", "healing_items")
  weight: number;       // Probability weight (0.0–1.0+, normalized by picker)
  count?: number;       // Stack count on spawn (default 1; Infinity for starting ammo)
  spawnSeparately?: boolean; // [special] Spawn multiple copies of item
}
```

### Example: `normal` Mode `ground_loot` Table

```typescript
ground_loot: [
  { table: "equipment", weight: 1 },
  { table: "healing_items", weight: 1 },
  { table: "ammo", weight: 1 },
  { table: "guns", weight: 0.9 },
  { table: "scopes", weight: 0.3 }
]
```

**Interpretation:** Total weight ≈ 5.2. When picking: equipment ~19%, healing ~19%, ammo ~19%, guns ~17%, scopes ~6%.

### Special Set-Count Crates

**Aegis Crate Example:**
```typescript
aegis_crate: {
  min: 3, max: 5,  // Picks 3–5 items total
  loot: [
    { table: "special_guns", weight: 1 },
    { table: "special_equipment", weight: 0.65 },
    { table: "special_scopes", weight: 0.3 },
    { table: "special_healing_items", weight: 0.15 }
  ]
}
```

Picks N items (3–5) from the pool, each weighted.

## Maps Data

### Registered Maps (11 Total)

#### Production Maps (6 Playable)

| Map | Dimensions | Mode | Special Features | Place Names |
|-----|-----------|------|---------|---------|
| **normal** | 1632 × 1632 | generic | Rivers (1–2), urban/industrial | Banana, Takedown, Lavlandet, Noskin Narrows, Mt. Sanger, Deepwood |
| **fall** | 1924 × 1924 | seasonal | Forest, 2 clearings, trails, pumpkins | Antler, Deadfall, Beaverdam, Crimson Hills, Emerald Farms, Darkwood |
| **halloween** | 1924 × 1924 | seasonal | Gothic/spooky, 3 clearings, graves | (6 themed places) |
| **infection** | 1632 × 1632 | seasonal | Corrupted/diseased biome | Blightnana, Quarantine, Rotlandet, Pathogen Narrows, Mt. Putrid, Decayedwood |
| **hunted** | 1924 × 1924 | survival | Wilderness, 1 centered river | Pine Cone, Gunpoint, Kuolema, Switchback, Black Hill, The Grove |
| **winter** | 1632 × 1632 | seasonal | Snow/ice, no rivers (special) | (6 themed places) |

#### Testing Maps (5 Debug)

| Map | Dimensions | Purpose |
|-----|-----------|---------|
| **debug** | 1620 × 1620 | Display all buildings + obstacles + items in grid |
| **arena** | 512 × 512 | Small multiplayer testing, fixed spawn |
| **singleBuilding** | 1024 × 1024 | Spawn one building in center (param: idString) |
| **singleObstacle** | 256 × 256 | Spawn one obstacle in center (param: idString) |
| **singleGun** | 256 × 256 | Spawn one gun + ammo in center (param: gun idString) |

### Map Structure Example

**normal map buildings:**
```typescript
buildings: {
  large_bridge: 2,
  small_bridge: Infinity,   // unlimited (spawns at every river)
  lighthouse: 1,
  port: 1,                  // major building
  warehouse: 5,
  // ... 30+ other types
}
```

**normal map obstacles:**
```typescript
obstacles: {
  oak_tree: 110,            // 110 oak trees (spawned randomly)
  regular_crate: 100,       // 100 loot crates
  grenade_crate: 35,        // 35 grenade crates
  rock: 150,
  bush: 110,
  barrel: 80,
  // ... more types
}
```

**normal map obstacle clumps:**
```typescript
obstacleClumps: [
  {
    clumpAmount: 100,       // 100 clumps of oak trees
    clump: {
      minAmount: 2,         // 2–3 trees per clump
      maxAmount: 3,
      jitter: 5,            // position variance (5 px)
      obstacles: ["oak_tree"],
      radius: 12            // cluster radius
    }
  },
  // ... more clumps (birch, pine, etc.)
]
```

### Map Features

**Rivers (normal map example):**
- 1–2 rivers per game (random)
- Obstacles: rocks (16 density), lily pads (6 density)
- Procedurally generated path, width 12–18 cells

**Clearings (fall map example, 2 total):**
- Dimensions: 200–250 px wide, 150–200 px tall
- Obstacles: boulders (3–6), flint crates (0–2), melee crates (0–1)
- Allowed obstacles: 11 types (clearing boulders, rocks, lilies, etc.)

**Trails (fall map example):**
- 2–3 trails per game
- Width: 2–4 cells (narrow) or 3–5 cells (wide, 20% chance)
- Obstacles: pebbles (300 density)

**Places (named zones):**
```typescript
places: [
  { name: "Banana", position: Vec(0.23, 0.2) },        // 23%, 20% of map
  { name: "Takedown", position: Vec(0.23, 0.8) }       // 23%, 80% of map
]
```

Used for radar labels, killfeeds, and player tracking.

## Default Inventory

### Generation Logic

Constructed at startup by iterating item definitions (@common/src/defaultInventory.ts):

```typescript
export const DEFAULT_INVENTORY: Record<string, number> = Object.create(null);

for (const item of [...HealingItems, ...Ammos, ...Scopes, ...Throwables]) {
    let amount = 0;
    switch (true) {
        case item.defType === DefinitionType.Ammo && item.ephemeral:
            amount = Infinity;     // Infinite ammo
            break;
        case item.defType === DefinitionType.Scope && item.giveByDefault:
            amount = 1;            // Hand scopes
            break;
    }
    DEFAULT_INVENTORY[item.idString] = amount;
}

Object.freeze(DEFAULT_INVENTORY);
```

### Result

- **Ephemeral ammo** (flag: `item.ephemeral`) → `Infinity` count (never depletes)
- **Default scopes** (flag: `item.giveByDefault`) → count `1`
- **All others** → count `0` (must be found in loot)

| Category | Example | Starting Count |
|----------|---------|-----------------|
| Ammo (ephemeral) | 9mm, rifle_ammo | ∞ |
| Scopes (default) | hand_scope | 1 |
| Healing items | Bandage, Medical Kit | 0 |
| Throwables | Frag Grenade, C4 | 0 |
| Armor/Backpack | (separate slots) | — |

### Ephemeral Ammo Mechanics

- Player spawns with `Infinity` count
- Picking up ammo: still `Infinity` (no increment)
- Firing weapon: consumes shots, stays `Infinity`
- Dies, respawns: resets to `Infinity`

Prevents "ammo hunting" for starter weapons; forces resource management for secondaries.

## Configuration Loading Pipeline

### Server Startup

```
1. server.ts: Start server
2. GameManager.init()
   → Import GasStages, LootTables, Maps (static constants)
   → Cache tables in memory (O(1) lookup)
3. new Game(mapName, teamMode)
   → Load Maps[mapName] definition
   → Map.onGenerate() hook (if custom)
   → Spawn buildings from maps[name].buildings
   → Spawn obstacles from maps[name].obstacles
   → Spawn loot from LootTables[modeName][ground_loot]
4. GasSystem.init()
   → Start at GasStages[0] (Inactive, no damage)
   → Schedule stage transitions @ stage.duration intervals
5. Player joins
   → Receive DEFAULT_INVENTORY in join packet
   → Receive map data + gas state in update
```

### Immutability

- All data is **frozen**: `Object.freeze(DEFAULT_INVENTORY)`
- Maps/stages are module constants (cannot be mutated)
- **No hot-reload:** Changes require server restart
- Performance: Lookups are O(1) (direct object access)

## Dependencies

### Depends On

- **[Object Definitions](../object-definitions/) — Item/Building/Obstacle registry** (@common/definitions/)
  - Loot table entries reference item `idString`
  - Building/obstacle entries look up their definition
  - Damage values validated against max health

### Depended On By

- **[Game Loop](../game-loop/) — Game initialization** (@server/src/game.ts)
  - On game creation, loads chosen map definition
  - Applies gas stage duration/damage timing

- **[Gas System](../gas/) — Gas progression** (@server/src/gas.ts)
  - Reads GasStages[currentStage] for radius, DPS, duration

- **[Map](../map/) — Map generation** (@server/src/map.ts)
  - Reads Maps[mapName] for obstacle/building counts
  - Reads LootTables[modeName] for ground loot distribution

- **[Inventory System](../inventory/) — Starting items** (@server/src/inventory/)
  - Initializes player inventory from DEFAULT_INVENTORY

## Known Issues & Gotchas

### 1. Gas Damage Unbounded

Final stage (11–13) applies 10 DPS indefinitely. Stage 12 lasts 60 seconds = 600 damage (most players ~100 HP).

**Mitigation:** None; intentional design. Players must reach final zone or die.

### 2. Loot Weight Normalization Not Enforced

No validation that weights sum to 1.0 or 100. Weights like `0.9` + `1.0` = 47% + 53% (not 90% + 100%).

**Prevention:** Test with logging or `validateDefinitions`.

### 3. Map Seed Deterministic

Same map definition = same layout (no randomized seed). Useful for testing, counterintuitive for procedural feel.

**Fix:** Implement optional `seed: number` field and randomize on live servers.

### 4. Container Spawn Hardcoded

Each map specifies obstacle/building counts manually (e.g., `regular_crate: 100`). No automatic density scaling.

**Pain:** Balancing across 6 maps is tedious; no adaptive spawning.

### 5. Default Inventory Global

DEFAULT_INVENTORY is applied to all players. No per-game or per-team variants.

**Limitation:** Custom game modes would require code changes, not config changes.

### 6. Obstacle Clumps vs. Individual Spawns Unclear

Both `obstacles` and `obstacleClumps` coexist. Unclear if `oak_tree: 110` is scattered or clumped (or mixed).

**Improvement:** Document the interaction or unify the API.

### 7. Loot Table Validation Missing

No check for item existence when loot tables reference items. Invalid `idString` could cause runtime errors.

**Fix:** Add validation in `validateDefinitions`.

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — System-wide design, subsystem interaction
- [Data Model](../../datamodel.md) — Entity definitions (maps, items, game objects)
- [API Reference](../../api-reference.md) — Packet protocol (DEFAULT_INVENTORY shipping)

### Tier 2 — Subsystems
- [Object Definitions](../object-definitions/README.md) — Item/building/obstacle registry
- [Game Loop](../game-loop/README.md) — Game initialization, gas stage timing
- [Gas System](../gas/README.md) — Gas damage application mechanics
- [Map](../map/README.md) — Map generation, biome systems, loot spawning
- [Inventory](../inventory/README.md) — Player inventory management
