# Map Generation — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/map/README.md -->

## Pattern: Loot Table Evaluation

**When to use:** Resolving which loot items spawn at a position (ground loot, building loot spawners, obstacle drops).

**Implementation:**

`getLootFromTable(modeName, tableID)` in `server/src/utils/lootHelpers.ts` dispatches against `LootTables[modeName][tableID]`.

A `LootTable` is one of three forms:

### Form 1 — Single-slot pick (flat `WeightedItem[]`)

One item is chosen via `weightedRandom()`. Each entry is either a direct item reference `{ item, weight }` or a recursive table reference `{ table, weight }`:

```typescript
ground_loot: [
    { table: "equipment",      weight: 1 },
    { table: "healing_items",  weight: 1 },
    { table: "ammo",           weight: 1 },
    { table: "guns",           weight: 0.9 },
    { table: "scopes",         weight: 0.3 }
]
```

### Form 2 — Guaranteed multi-slot pick (`WeightedItem[][]`)

Each inner array is resolved independently — one pick per slot. Used for containers that always produce N items (one from each slot):

```typescript
viking_chest: [
    [{ item: "seax", weight: 1 }],                         // slot 1: always seax
    [{ table: "viking_chest_guns", weight: 1 }],           // slot 2: a gun
    [{ table: "viking_chest_guns", weight: 1 }],           // slot 3: another gun
    [{ table: "special_equipment", ... }, { table: "special_scopes", ... }], // slot 4
]
```

### Form 3 — Variable-count pick (`FullLootTable`)

Rolls `random(min, max)` times over the `loot` array:

```typescript
aegis_crate: {
    min: 3,
    max: 5,
    loot: [
        { table: "special_guns",           weight: 1 },
        { table: "special_equipment",      weight: 0.65 },
        { table: "special_scopes",         weight: 0.3 },
        { table: "special_healing_items",  weight: 0.15 }
    ]
}
```

The optional `noDuplicates: true` field prevents the same item from being drawn twice from one evaluation.

**Example files:** `@file server/src/data/lootTables.ts`, `@file server/src/utils/lootHelpers.ts`

---

## Pattern: MapDefinition Registration

**When to use:** Adding a new permanent game map or a new dev/test map.

**Implementation:**

Add a new key to the `maps` object literal in `server/src/data/maps.ts`. The object is typed as `satisfies Record<string, MapDefinition>`, so TypeScript enforces all required fields. The key is automatically added to the `MapName` union type via `keyof typeof maps`.

```typescript
// server/src/data/maps.ts

const maps = {
    // ... existing maps ...

    my_new_map: {
        width: 1632,
        height: 1632,
        oceanSize: 128,
        beachSize: 32,

        // Optional: override game mode
        mode: "normal",

        // Rivers and trails
        rivers: {
            minAmount: 1, maxAmount: 2,
            maxWideAmount: 1, wideChance: 0.35,
            minWidth: 12, maxWidth: 18,
            minWideWidth: 25, maxWideWidth: 30,
            obstacles: { river_rock: 16 }
        },

        // Major buildings — one per quadrant max, widely spaced
        majorBuildings: ["port", "headquarters"],

        // Regular buildings — count per map; use Infinity for bridges
        buildings: {
            small_bridge: Infinity,
            warehouse: 4
        },

        // Per-quadrant caps for regular buildings
        quadBuildingLimit: {
            warehouse: 2
        },

        // Standalone obstacles — count per map
        obstacles: {
            oak_tree: 100,
            rock: 150,
            regular_crate: 80
        },

        // Obstacle clumps
        obstacleClumps: [{
            clumpAmount: 50,
            clump: { obstacles: ["oak_tree"], minAmount: 2, maxAmount: 3, radius: 12, jitter: 5 }
        }],

        // Ground loot (references LootTables key)
        loots: { ground_loot: 60 },

        // Named locations (fractional 0.0–1.0 coordinates)
        places: [
            { name: "My Town", position: Vec(0.25, 0.25) }
        ]
    }
} satisfies Record<string, MapDefinition>;
```

To use the new map in a game, pass `"my_new_map"` as the `map` argument when creating a `Game` instance (or configure it in `server/config.json`).

**Example files:** `@file server/src/data/maps.ts`

---

## Pattern: Custom Generation via `onGenerate`

**When to use:** A map needs logic that the declarative `MapDefinition` fields cannot express — e.g., programmatic building counts, forcing a building at an exact position, or spawning loot from code rather than tables.

**Implementation:**

Define `onGenerate(map: GameMap, params: string[])` in the `MapDefinition`. It is called as the **last generation step**, after all declarative placements.

The `params` array is populated by splitting the map string on `:`:  
`"singleBuilding:headquarters"` → `params = ["headquarters"]`.

```typescript
my_map: {
    width: 1024,
    height: 1024,
    oceanSize: 64,
    beachSize: 32,

    onGenerate(map, params) {
        // Force a building at the exact center
        map.generateBuilding("headquarters", Vec(this.width / 2, this.height / 2), 0);

        // Programmatic loot
        map.game.addLoot("mcx_spear", Vec(512, 512), 0, { count: 1, jitterSpawn: false });

        // Programmatic obstacles
        for (let i = 0; i < 10; i++) {
            const position = map.getRandomPosition(
                new CircleHitbox(5),
                { spawnMode: MapObjectSpawnMode.Grass }
            );
            if (position) map.generateObstacle("rock", position);
        }
    }
}
```

Internally used by: `nye`, `debug`, `arena`, `singleBuilding`, `singleObstacle`, `singleGun`, `gunsTest`, `gallery`.

**Example files:** `@file server/src/data/maps.ts`, `@file server/src/map.ts`

---

## Pattern: Obstacle Clump Placement

**When to use:** Spawning naturally grouped obstacles (tree groves, rock clusters) instead of uniformly random placement.

**Implementation:**

Use `MapDefinition.obstacleClumps`. Each entry in the array defines one **clump type**:

- `clumpAmount` — how many clump centers are seeded on the map.
- `clump.radius` — arc radius; obstacles are placed at equal angular steps around the center.
- `clump.jitter` — random noise added to each obstacle position (keeps groups from looking too artificial).
- `clump.obstacles` — list of eligible obstacle `idString`s; one is chosen randomly per obstacle slot.

```typescript
obstacleClumps: [
    {
        clumpAmount: 100,               // 100 grove centers
        clump: {
            obstacles: ["oak_tree"],
            minAmount: 2,
            maxAmount: 3,
            radius: 12,                  // ~12 units from center
            jitter: 5                    // ±5 units random nudge
        }
    },
    {
        clumpAmount: 25,
        clump: {
            obstacles: ["birch_tree"],
            minAmount: 2,
            maxAmount: 3,
            radius: 12,
            jitter: 5
        }
    }
]
```

All counts are scaled by `mapScale()` if the map's `scale` option is not 1.

**Example files:** `@file server/src/map.ts`, `@file server/src/data/maps.ts`

---

## Pattern: River Inline Obstacle Density

**When to use:** Placing obstacles that should appear **inside** river or trail areas at a density proportional to the river size.

**Implementation:**

Obstacles listed in `RiverDefinition.obstacles` use a density formula rather than a fixed count:

```
amount = defined_density * river.width * river.points.length / 500
```

The magic divisor `500` normalizes the density coefficient so that map authors can write human-readable numbers (e.g., `river_rock: 16`) rather than fractions:

```typescript
rivers: {
    obstacles: {
        river_rock: 16,   // ~16-unit density → produces ~N rocks per river
        lily_pad: 6
    }
}
```

Obstacles that should spawn **on the bank** must have `spawnMode: MapObjectSpawnMode.River` in their `ObstacleDefinition`. Obstacles with `spawnMode: MapObjectSpawnMode.Trail` are placed along trails. Both are placed via `river.getRandomPosition()`, which samples uniformly along the spline.

**Example files:** `@file server/src/map.ts` (_generateRiverObstacles), `@file server/src/data/maps.ts`

---

## Pattern: Seeded River Paths

**When to use:** Understanding why river shapes are reproducible between server and client, or why other map elements are not.

**Implementation:**

River path walking uses `new SeededRandom(this.seed)` where `this.seed` was generated by the unseeded `random(0, 2**31)`. The seed is written into `MapData.seed` (uint32) and transmitted to the client in the `MapPacket`.

The client's `Terrain` and `River` objects are reconstructed from the serialized points (not re-generated from the seed); the seed is included in the packet for display purposes and potential future use.

All other generation (building positions, obstacle positions, loot) uses unseeded `random()` / `randomFloat()` / `pickRandomInArray()` — these are **not reproducible** from the seed.

**Example files:** `@file server/src/map.ts` (constructor), `@file common/src/utils/random.ts`

---

## Pattern: Spawn Mode Selection

**When to use:** Choosing where an obstacle or building is allowed to appear.

**Implementation:**

The `spawnMode` field on an `ObstacleDefinition` or `BuildingDefinition` tells `getRandomPosition()` which region to sample from. The default is `MapObjectSpawnMode.Grass`.

| Mode | Valid positions | Typical use |
|------|----------------|-------------|
| `Grass` | Inland grass, inside `beachPadding` | Trees, crates, rocks |
| `GrassAndSand` | Inland + beach strip | Ground loot drops |
| `River` | Inside non-trail river water hitboxes | Water crates, lily pads |
| `Trail` | On trail bank | Trail rocks, pebbles |
| `Beach` | One of the 4 beach rectangles | Lighthouses, buoys, tugboats |
| `Ring` | Circle of fixed radius around the map center | Symmetrical ring spawns |

Clearings punch holes in the `Grass` zone: obstacles with `ignoreClearings: false` (the default) are rejected if their chosen position falls inside a clearing hitbox.

**Example files:** `@file server/src/map.ts` (getRandomPosition), `@file common/src/constants.ts` (MapObjectSpawnMode enum)
