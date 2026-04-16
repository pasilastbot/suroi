# Map Generation Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/map/modules/ -->
<!-- @source: server/src/map.ts, server/src/data/maps.ts -->

## Purpose

`GameMap` procedurally generates the full game world inside its constructor at game startup: placing buildings, obstacles, rivers, trails, clearings, and ground loot, then serializing the static result once into a cached binary `MapPacket` buffer that is sent to every player who joins.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/map.ts` | `GameMap` class — all procedural generation logic |
| `server/src/data/maps.ts` | `MapDefinition` interface, `RiverDefinition` interface, `ObstacleClump` interface, and the `Maps` registry |
| `common/src/utils/terrain.ts` | `Terrain` and `River` classes — flood/floor lookup grid, Catmull-Rom river splines |
| `server/src/data/lootTables.ts` | `LootTables` — per-mode weighted loot table definitions |
| `server/src/utils/lootHelpers.ts` | `getLootFromTable()`, `LootTable` type, `LootItem` type |
| `common/src/packets/mapPacket.ts` | `MapPacket`, `MapData` — binary serialization format sent to clients |

## Architecture

Generation happens entirely in the `GameMap` constructor. There is no deferred or incremental generation — the world is fully built before the constructor returns. The result is immediately serialized into `this.buffer: ArrayBuffer`.

The `GameMap` has **two sources of randomness**:

1. **Seeded (`SeededRandom`)** — used only for river and trail path walking, so that given the same `seed` value, the exact river paths can be reproduced. The seed is always transmitted to the client inside `MapPacket`.
2. **Unseeded (`random()`, `randomFloat()`, etc.)** — used for building placement, obstacle placement, loot spawning, and everything else. These produce different results each game even with the same seed.

The seed itself (`this.seed`) is generated with unseeded `random(0, 2**31)`.

### MapOptions

```typescript
export interface MapOptions {
    scale?: number           // Multiplies width/height and scales all placement counts
    maxMajorBuildings?: number  // Caps how many major buildings spawn (for reduced-scale games)
    gameSpawnWindow?: number    // Overrides GAME_SPAWN_WINDOW
}
```

When `scale !== 1` the `mapScale()` helper proportionally scales all counts as:
`Math.round(count * (width * scale)² / width²)`.

### How a Game Constructs Its Map

In `server/src/game.ts` line 259:

```typescript
this.map = new GameMap(this, map, mapOptions);
```

`map` is a string in the format `"mapName"` or `"mapName:param1:param2"`. The `params` array after splitting on `:` is forwarded to `MapDefinition.onGenerate()`.

## `MapDefinition` Interface

Defined in `server/src/data/maps.ts`:

```typescript
export interface MapDefinition {
    readonly width: number
    readonly height: number
    readonly mode?: ModeName         // Force a specific game mode for this map
    readonly spawn?: SpawnOptions    // Player spawn mode (default: Normal)
    readonly oceanSize: number       // Width of the ocean border in units
    readonly beachSize: number       // Width of the sand beach strip in units

    readonly rivers?: RiverDefinition
    readonly trails?: RiverDefinition   // Narrower, trail variant (no waterHitbox)

    readonly clearings?: {
        readonly minWidth: number
        readonly minHeight: number
        readonly maxWidth: number
        readonly maxHeight: number
        readonly count: number
        readonly allowedObstacles: readonly ReferenceTo<ObstacleDefinition>[]
        readonly obstacles: ReadonlyArray<{
            readonly idString: ReferenceTo<ObstacleDefinition>
            readonly min: number   // minimum amount per clearing
            readonly max: number   // maximum amount per clearing
        }>
    }

    readonly bridges?: readonly ReferenceTo<BuildingDefinition>[]
    readonly majorBuildings?: readonly ReferenceTo<BuildingDefinition>[]
    readonly buildings?: Record<ReferenceTo<BuildingDefinition>, number>
    readonly quadBuildingLimit?: Record<ReferenceTo<BuildingDefinition>, number>
    readonly obstacles?: Record<ReferenceTo<ObstacleDefinition>, number>
    readonly obstacleClumps?: readonly ObstacleClump[]
    readonly loots?: Record<keyof typeof LootTables[ModeName], number>

    readonly places?: ReadonlyArray<{
        readonly name: string
        readonly position: Vector   // FRACTIONAL: 0.0–1.0 of map width/height
    }>

    readonly onGenerate?: (map: GameMap, params: string[]) => void
}
```

### `RiverDefinition` Interface

```typescript
export interface RiverDefinition {
    readonly minAmount: number
    readonly maxAmount: number
    readonly maxWideAmount: number   // Cap on how many rivers can be "wide"
    readonly wideChance: number      // Per-river probability of being wide
    readonly minWidth: number
    readonly maxWidth: number
    readonly minWideWidth: number
    readonly maxWideWidth: number
    readonly obstacles: Record<ReferenceTo<ObstacleDefinition>, number>
    // Density formula: count * riverWidth * numPoints / 500
    readonly centered?: boolean      // Lock river to straight pass through center
}
```

### `ObstacleClump` Interface

```typescript
export interface ObstacleClump {
    readonly clumpAmount: number   // Number of clumps on the full map
    readonly clump: {
        readonly obstacles: readonly ReferenceTo<ObstacleDefinition>[]
        readonly minAmount: number   // Obstacles per clump
        readonly maxAmount: number
        readonly radius: number      // Arc radius for placed obstacles
        readonly jitter: number      // Random positional offset per obstacle
    }
}
```

## Available Maps

All keys in the `Maps` registry from `server/src/data/maps.ts`:

### Production / Gameplay Maps

| Key | Width × Height | Notes |
|-----|---------------|-------|
| `normal` | 1632 × 1632 | Standard summer map; major buildings: `port`, `headquarters`, `armory`, `refinery` |
| `fall` | 1924 × 1924 | Autumn theme; trails + clearings; major: `lodge`, `campsite`, `bombed_armory` |
| `halloween` | 1924 × 1924 | Halloween theme; pumpkins/plumpkins/gravestones; major: `armory`, `lodge`, `headquarters`, `refinery` |
| `infection` | 1632 × 1632 | Infection mode variant of normal; adds `graveyard`, `medical_camp` |
| `hunted` | 1924 × 1924 | Hunted mode; single centered river; major: `sawmill`, `shooting_range`, `tavern` |
| `winter` | 1632 × 1632 | Winter/ice theme; winter variants of all obstacles; major: `port`, `headquarters`, `armory`, `refinery` |

### Event Maps

| Key | Width × Height | Notes |
|-----|---------------|-------|
| `nye` | 1452 × 1452 | New Year's Eve; custom logic in `onGenerate`; `christmas_camp` forced at center |

### Development / Testing Maps

| Key | Width × Height | Notes |
|-----|---------------|-------|
| `debug` | 1620 × 1620 | Renders all buildings, obstacles, and loot items in grid layout |
| `arena` | 512 × 512 | Fixed-spawn test arena; all loot items are spawned in 4 quadrants |
| `singleBuilding` | 1024 × 1024 | Spawns one building by `params[0]` at center |
| `singleObstacle` | 256 × 256 | Spawns one obstacle by `params[0]` at center |
| `singleGun` | 256 × 256 | Spawns one gun + infinite ammo at center |
| `gunsTest` | Dynamic | Bot-driven gunplay test; dynamically sized to fit all gun rows |
| `obstaclesTest` | 128 × 128 | Fills map with one obstacle type in a grid |
| `playersTest` | 256 × 256 | Populates map with random-placed barrel or player slots |
| `lootTest` | 256 × 256 | Spawns all armor/backpack loot items in a row |
| `bunkerSpawnTest` | 1024 × 1024 | 150 small bunkers to stress-test bunker placement |
| `river` | 1344 × 1344 | Empty map with no buildings/obstacles; river path testing |
| `armory` | 850 × 850 | Single armory building + scattered obstacles |
| `new_port` | 875 × 875 | Single port building + scattered obstacles |
| `gallery` | 1024 × 1024 | Custom `onGenerate`; headquarters at center + random surroundings |

## Map Coordinate Space

- All coordinates are in **game units** (not pixels).
- `GameConstants.maxPosition` = **1924** — the largest map dimension in any current map.
- `GameConstants.gridSize` = **32** — the spatial grid cell size used by `Grid`.
- `Terrain` uses an internal cell size of **64** units for its floor/river lookup grid.
- Origin (0, 0) is top-left. The ocean ring occupies `0` to `oceanSize` on all edges.
- Place names use **fractional positions** (0.0–1.0) which are multiplied by `width`/`height` and jittered by ±4% at generation time.

## Generation Algorithm

The following steps occur in order inside `GameMap`'s constructor:

| Step | What Happens |
|------|-------------|
| 1 | Parse `mapData` string: `"name:param1:param2"` → `name` + `params[]` |
| 2 | Initialize `mapScale()` closure (identity if `scale === 1`, otherwise quadratic scaling) |
| 3 | Generate seed: `random(0, 2**31)`, write to `MapData` packet |
| 4 | Set `width = mapDef.width * scale`, `height = mapDef.height * scale` |
| 5 | Build `beachHitbox` (4-part `GroupHitbox`) and `islandHitbox` (`RectangleHitbox`) |
| 6 | Generate trails (if `mapDef.trails`) using `SeededRandom(seed)` |
| 7 | Generate rivers (if `mapDef.rivers`) using same `SeededRandom` instance |
| 8 | Construct `Terrain` with the river list (jagged beach/grass polygon + floor/river lookup grid) |
| 9 | Place **major buildings** (`mapDef.majorBuildings`) — 1 per quadrant, min 150 000 units² spacing between any two |
| 10 | Place **regular buildings** (`mapDef.buildings`) — respecting `quadBuildingLimit` per quadrant; bridges placed along rivers separately |
| 11 | Generate **clearings** (`mapDef.clearings`) — rectangular open zones with controlled obstacle populations |
| 12 | Spawn **river inline obstacles** (`mapDef.rivers.obstacles`) using density formula |
| 13 | Spawn **trail inline obstacles** (`mapDef.trails.obstacles`) using density formula |
| 14 | Spawn **obstacle clumps** (`mapDef.obstacleClumps`) — circular groupings of 2–N obstacles |
| 15 | Spawn **standalone obstacles** (`mapDef.obstacles`) |
| 16 | Spawn **ground loot** (`mapDef.loots`) — via loot table system, spawning on `GrassAndSand` positions |
| 17 | Call **`onGenerate`** callback (custom generation hook, used by `nye`, `debug`, `arena`, etc.) |
| 18 | Jitter and serialize **place names** (±4% of map dimensions) |
| 19 | Serialize full `MapData` packet → `this.buffer: ArrayBuffer` |

## Building Placement

Buildings fall into two categories with different placement logic:

### Regular Buildings

A non-bridge building attempts up to **100 outer attempts**, each calling `getRandomPosition()` with up to **400 inner attempts**.

The map is divided into 4 **quadrants**:

```
(0,0)───────(width/2)───────(width)
  │                            │
  │     Q1        Q2           │
(height/2)                     │
  │     Q3        Q4           │
  │                            │
(height)────────────────────(width,height)
```

- `majorBuildings`: at most **1 per quadrant**; any two major buildings must be more than `√150 000 ≈ 387` units apart.
- `quadBuildingLimit`: any building type listed here is capped at the given count per quadrant (e.g., `warehouse: 2` means no more than 2 warehouses in any single quadrant).

### Bridge Buildings

Bridges (`buildingDef.bridgeHitbox` is non-null) are placed **along rivers**, not via `getRandomPosition()`. The algorithm:

1. Iterate all non-trail rivers wide enough (`river.width >= bridgeMinRiverWidth`).
2. For each river, scan positions from `t = 0.05` to `t = 0.95` (parameterized along the spline).
3. Score each position by how closely its tangent direction aligns to a cardinal axis (orientation 0–3).
4. Place the bridge at the position with minimal angle deviation that passes all spawn-hitbox collision checks.
5. `bridgeSpawnRanges` (default `[[0.1, 0.4], [0.6, 0.9]]`) restricts bridge positions to the first and second thirds of each river, shuffled randomly.

### Spawn Modes

`getRandomPosition()` honors the obstacle/building definition's `spawnMode`:

| `MapObjectSpawnMode` | Where positions are sampled |
|----------------------|----------------------------|
| `Grass` | Inside beach padding (default for most objects) |
| `GrassAndSand` | Inside ocean padding (wider area; used by ground loot) |
| `River` | Random point on a random non-trail river's water hitbox |
| `Trail` | Random point on a random trail |
| `Beach` | Random point on one of the 4 beach rectangles |
| `Ring` | Point on circle of `spawnRadius` centered on map center |

## Obstacle Placement

Standalone obstacles from `mapDef.obstacles` are placed by `_generateObstacles()`:

- Count is scaled by `mapScale()`.
- Per-obstacle: random scale in `[spawnMin, spawnMax]`, random variation index, random rotation per `rotationMode`.
- `getRandomPosition()` is called up to 200 attempts; on failure a warning is logged and the instance is skipped.
- Obstacles flagged `ignoreClearings: true` (those in `clearings.allowedObstacles`) may spawn inside clearing zones; all others are excluded.

### Obstacle Clumps

`_generateObstacleClumps()` places groups of 2–N obstacles in a circular arc pattern:

1. Find a center position using `getRandomPosition()`.
2. Pick `random(minAmount, maxAmount)` obstacles.
3. Place each obstacle at angle `j * (2π / count) + randomOffset`, displaced by `radius` from center, with additional `jitter` noise.
4. All clump obstacles are picked randomly from `clump.obstacles[]`.

## River Generation

Rivers and trails share the same `_generateRivers()` / `_generateRiver()` code path, with `isTrail` toggling behavior:

### Phase 1 — Count and Width Selection

1. `amount = SeededRandom.getInt(minAmount, maxAmount)` (seeded, scaled by `mapScale()`).
2. For each river: sample a width in `[minWidth, maxWidth]`; upgrade to `[minWideWidth, maxWideWidth]` if `wideAmount < maxWideAmount && random() < wideChance`.
3. Sort widths descending — wide rivers generate first, reducing collision interference.

### Phase 2 — Path Walking (per river)

1. Pick a random start point on a map-edge padding line (horizontal or vertical, forward or reversed). For `centered` rivers, start exactly at the midpoint of the edge.
2. Compute start angle toward map center (±π based on direction).
3. Walk **60 points** (rivers) or **25 points** (trails), each step:
   - Compute `distFactor = distance(lastPoint, center) / (width / 2)`.
   - `maxDeviation = lerp(0.8, 0.1, distFactor)` — path naturally straightens near center.
   - New angle = previous angle + random deviation in `[-maxDeviation, +maxDeviation]`.
   - New position = `lastPoint + fromPolar(angle, random(30, 80))`.
   - **Terminate early** if the new point intersects another river/trail path (collision check against all previous river segments), or exits map bounds (clamp and stop).
4. **Reject** the river if `riverPoints.length < 20` or `> 59`.

### Phase 3 — River Object Construction

`new River(width, points, otherRivers, mapBounds, isTrail)`:

- `bankWidth = clamp(width * 0.75, 12, 20)` for rivers; `= width` for trails.
- `waterHitbox`: `PolygonHitbox` built from `±width` normal offsets at each point (rivers only; trails have no `waterHitbox`).
- `bankHitbox`: `PolygonHitbox` from `±(width + bankWidth)` offsets.
- River width **increases near map bounds** via `(1 + end³ × 1.5) × width`.
- Spline interpolation via **Catmull-Rom** (`getPosition(t)`, `getTangent(t)`, `getNormal(t)`).

### Inline River Obstacles

After the `Terrain` is built, `_generateRiverObstacles()` spawns obstacles inside each river:

```
amount = obstacles[idString] * river.width * river.points.length / 500
```

Obstacles with `spawnMode === MapObjectSpawnMode.Trail` are placed on the trail bank; others on the river's water/bank surface.

## Loot Spawning

Ground loot is spawned from `mapDef.loots`, which maps loot table names to spawn counts:

```typescript
mapDef.loots = { ground_loot: 60 }  // spawn 60 picks from "ground_loot" table
```

Each call to `_generateLoots(table, count)`:
1. Calls `getLootFromTable(modeName, table)` to evaluate the loot table.
2. Finds a random `GrassAndSand` position using a `CircleHitbox(5)`.
3. Calls `game.addLoot()` for each resolved `LootItem`.

Building loot spawners (defined in `BuildingDefinition.lootSpawners`) are triggered at building instantiation time via `generateBuilding()` — not via `mapDef.loots`.

### Loot Table Format

From `server/src/utils/lootHelpers.ts`:

```typescript
type LootTable = SimpleLootTable | FullLootTable;

// Simple: array of weighted entries (one pick total)
type SimpleLootTable = readonly WeightedItem[];
// OR: array of arrays (one pick per inner array = guaranteed multi-slot)
type SimpleLootTable = ReadonlyArray<readonly WeightedItem[]>;

interface FullLootTable {
    readonly min: number       // minimum items to generate
    readonly max: number       // maximum items to generate
    readonly noDuplicates?: boolean
    readonly loot: readonly WeightedItem[]
}

type WeightedItem =
    | { readonly item: idString, readonly weight: number }   // direct item reference
    | { readonly table: string, readonly weight: number }    // recursive table reference
```

`getLootFromTable(modeName, tableID)` resolves the table from `LootTables[modeName][tableID]` and evaluates `weightedRandom()` on the entries.

## MapPacket

From `common/src/packets/mapPacket.ts`:

```typescript
interface MapData {
    readonly type: PacketType.Map
    readonly seed: number          // uint32
    readonly width: number         // uint16
    readonly height: number        // uint16
    readonly oceanSize: number     // uint16
    readonly beachSize: number     // uint16
    readonly rivers: Array<{
        readonly width: number     // uint8
        readonly points: readonly Vector[]
        readonly isTrail: boolean
    }>
    readonly objects: MapObject[]   // buildings + visible obstacles
    readonly places: ReadonlyArray<{ readonly position: Vector, readonly name: string }>
}
```

**Serialization encoding:**
- Obstacle rotation and variation are bit-packed into a single `uint8` to save bandwidth: variation occupies the MSBs, rotation occupies the LSBs (for `Limited`/`Binary` modes).
- `Building` objects encode orientation (2 bits via `writeObstacleRotation`) and layer.

**Caching:** The packet is serialized once in the constructor via `PacketStream` → `this.buffer: ArrayBuffer`. No per-player re-serialization. The server writes this buffer directly to each joining player's WebSocket.

## Data Flow

```
Game constructor
  └─ new GameMap(game, "normal", options)
       ├─ seed = random(0, 2^31)
       ├─ SeededRandom(seed) → trails → rivers
       ├─ new Terrain(width, height, oceanSize, beachSize, seed, rivers)
       ├─ majorBuildings → _generateBuildings()  ──→ game.grid.addObject(building)
       ├─ buildings      → _generateBuildings()  ──→ game.grid.addObject(building)
       ├─                  bridges via river splines
       ├─ clearings      → _generateClearings()
       ├─ river/trail obstacles → _generateRiverObstacles()
       ├─ obstacleClumps → _generateObstacleClumps()
       ├─ obstacles      → _generateObstacles() ───→ game.grid.addObject(obstacle)
       ├─ loots          → _generateLoots() ───────→ game.addLoot()
       ├─ mapDef.onGenerate?.()
       └─ PacketStream.serialize(MapData) → this.buffer: ArrayBuffer

Player joins game
  └─ server writes this.buffer → player WebSocket (no re-serialization)
```

## Plugin Events

`GameMap` fires two `PluginManager` events during generation (from `server/src/pluginManager.ts`):

| Event | Fired When | Return `true` to skip |
|-------|-----------|----------------------|
| `building_will_generate` | Before instantiating a `Building` | Yes |
| `building_did_generate` | After building is added to grid | No |
| `obstacle_will_generate` | Before instantiating an `Obstacle` | Yes |
| `obstacle_did_generate` | After obstacle is added to grid | No |

## Interfaces & Contracts

| Export | Location | Description |
|--------|----------|-------------|
| `GameMap` | `server/src/map.ts` | Main class; owns the generated world state |
| `MapOptions` | `server/src/map.ts` | Constructor options (`scale`, `maxMajorBuildings`, `gameSpawnWindow`) |
| `MapDefinition` | `server/src/data/maps.ts` | Config shape for a single map |
| `RiverDefinition` | `server/src/data/maps.ts` | River/trail config |
| `ObstacleClump` | `server/src/data/maps.ts` | Clump placement config |
| `Maps` | `server/src/data/maps.ts` | Registry of all maps: `Record<MapName, MapDefinition>` |
| `MapName` | `server/src/data/maps.ts` | Union of all valid map key strings |
| `MapData` | `common/src/packets/mapPacket.ts` | Packet payload shape |
| `LootTable` | `server/src/utils/lootHelpers.ts` | `SimpleLootTable \| FullLootTable` |
| `Terrain` | `common/src/utils/terrain.ts` | Floor/river lookup grid |
| `River` | `common/src/utils/terrain.ts` | River spline object with polygon hitboxes |

## Module Index (Tier 3)

For procedural generation algorithm details, see:

- [Generation Algorithm Module](modules/generation-algorithm.md) — Step-by-step map generation sequence, placement strategies, terrain generation

## Known Gotchas

- **`places` positions are fractional**, not absolute. `{ position: Vec(0.23, 0.2) }` is later multiplied by `this.width`/`this.height` with ±4% jitter. Do not use raw game units there.
- **River generation is seeded; everything else is not.** Two games with the same seed will have identical river paths but different building/obstacle layouts.
- **`Infinity` as building count** is used for `small_bridge` — bridges are placed algorithmically along rivers until all river ranges are exhausted; the `count` only acts as an upper bound in the bridge loop.
- **`mapScale()` rounds to minimum 1.** If `scale` is very small, counts that would compute to 0 are raised to 1.
- **`clearings.allowedObstacles`** controls which obstacle types are exempt from clearing exclusion zones. Obstacles NOT in this list are forbidden from spawning inside any clearing's hitbox.
- **`mode` field in `MapDefinition`** is not enforced by `GameMap` itself — it is used by `modeFromMap()` in game setup to select the game mode before the map is constructed.

## Dependencies

- **Depends on:**
  - [Object Definitions](../object-definitions/) — `Buildings`, `Obstacles` registries for `reify()` and definition lookup
  - [Networking](../networking/) — `MapPacket`, `PacketStream` for buffer serialization
  - [Spatial Grid](../spatial-grid/) — `game.grid.addObject()` called for every placed building and obstacle
  - `common/src/utils/random.ts` — `SeededRandom`, `random()`, `randomFloat()`, `randomVector()`, `pickRandomInArray()`
  - `common/src/utils/hitbox.ts` — `RectangleHitbox`, `CircleHitbox`, `GroupHitbox`, `PolygonHitbox`
  - `server/src/pluginManager.ts` — plugin event bus for `building_will_generate` etc.

- **Depended on by:**
  - [Game Loop](../game-loop/) — `Game` constructor creates `GameMap`; tick loop reads `game.map` for terrain queries
  - [Gas System](../gas/) — reads `game.map.width`/`height` to compute gas circle bounds
  - Server player join path — writes `game.map.buffer` directly to each connecting player's WebSocket

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview
- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — Core entity model
- **Tier 2:** [Object Definitions](../object-definitions/) — Building and obstacle definition registries
- **Tier 2:** [Networking](../networking/) — MapPacket delivery and binary protocol
- **Tier 2:** [Spatial Grid](../spatial-grid/) — Grid that holds all placed game objects
- **Tier 2:** [Game Loop](../game-loop/) — Game that owns and creates this map
- **Patterns:** [patterns.md](patterns.md) — Reusable map generation patterns
