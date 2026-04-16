# Map Generation Algorithm Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/map/README.md -->
<!-- @source: server/src/map.ts -->

## Purpose

`GameMap` runs a deterministic, seeded procedural generation pipeline that creates rivers, terrain, buildings, obstacles, clearings, and ground loot for a single game session, then caches the result as a serialized `ArrayBuffer` sent to every joining player.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/map.ts` | `GameMap` class — full generation pipeline | High |
| `common/src/utils/terrain.ts` | `Terrain` and `River` classes — polygon geometry and floor grid | High |
| `common/src/utils/random.ts` | `SeededRandom` LCG and unseeded helpers | Medium |
| `server/src/data/maps.ts` | `MapDefinition` and `RiverDefinition` interfaces; `Maps` record | Medium |

## Business Rules

- **Seed is generated with `random(0, 2 ** 31)` — not SeededRandom.** The seed itself is produced by the unseeded (Math.random-based) `random()`, then written into the `MapPacket`. `SeededRandom` is only instantiated for river/trail generation, using that already-random seed.
- **`mapData` is a colon-separated string.** `GameMap` receives `mapData: string` and splits it as `[name, ...params]`; the first segment is the `MapName` key into `Maps`, the rest are passed as `params[]` to `mapDef.onGenerate`.
- **`mapScale` function normalises counts by area.** When `scale !== 1`, the helper function scales each count by `(mapDef.width * scale)² / mapDef.width²`. At `scale = 1`, it is the identity function to avoid unnecessary multiplication.
- **Quadrant system for major buildings.** The map is divided into 4 quadrants by the midpoint `(width/2, height/2)`. `quadMajorBuildings` tracks which quadrants already have a major building. A new major building is rejected if its quadrant already contains one, or if any existing major building is within distance² of 150,000 units.
- **`quadBuildingLimit` caps per-quadrant count for non-major buildings.** When `idString in quadBuildingLimit`, placement is rejected if that building type already appears `>= quadBuildingLimit[idString]` times in the target quadrant.
- **Bridges are placed along river splines, not on the grid.** If a `BuildingDefinition` has a `bridgeHitbox`, placement iterates `terrain.rivers` and tries fractional `t` positions (`[0.1, 0.4]` and `[0.6, 0.9]` by default, order shuffled with `Math.random`), picking the orientation with the smallest angular difference from the river tangent.
- **River obstacle count is density-based.** The formula is `obstacleRecord[idString] * river.width * river.points.length / 500`. The `/500` divisor exists so that definition authors can write human-scale integers rather than tiny decimals.
- **`onGenerate` is called after all automatic placement**, receiving `(map, params)` where `params` are the colon-split segments from `mapData`. It can call any `GameMap` public method.
- **Named places have ±4% position jitter.** Each entry in `mapDef.places` is an (name, relative position) pair; absolute position is `width * (x ± randomFloat(-0.04, 0.04))`, adding mild visual variation.
- **The map packet buffer is serialized once and cached.** `this.buffer` is an `ArrayBuffer` built by `PacketStream` at the end of the constructor. Every joining player receives this same buffer via the join flow.

## Generation Pipeline (Detailed)

### Step 1: Parse `mapData` and resolve `MapDefinition`

```typescript
const [name, ...params] = mapData.split(":") as [MapName, ...string[]];
const mapDef: MapDefinition = Maps[name];
```

`params` is stored and passed to `onGenerate` at the end of the pipeline.

### Step 2: Scale setup

```typescript
const { scale = 1, maxMajorBuildings } = options;
this.mapScale = scale === 1
    ? (num) => num
    : (num) => Math.round(num * ((mapDef.width * scale) ** 2 / mapDef.width ** 2));
```

Dimensions are then:
```typescript
this.width  = mapDef.width  * scale;
this.height = mapDef.height * scale;
```

When `scale !== 1`, any `mapDef.quadBuildingLimit` counts are also scaled.

### Step 3: Seed generation

```typescript
this.seed = packet.seed = random(0, 2 ** 31);
```

`random()` is the unseeded `Math.random`-based helper from `common/src/utils/random.ts`. The seed is written into the `MapPacket` so clients can reconstruct terrain client-side with the same `Terrain` constructor.

### Step 4: Beach and island hitboxes

```typescript
const beachPadding = mapDef.oceanSize + mapDef.beachSize + Numeric.min(mapDef.oceanSize + mapDef.beachSize, 8);
const oceanSize    = mapDef.oceanSize + Numeric.min(mapDef.oceanSize, 8);
```

The `+ Numeric.min(…, 8)` on each adds up to 8 extra units to account for jagged polygon vertex offsets.

`beachHitbox` is a `GroupHitbox` of **4 `RectangleHitbox`es** forming the four sides of the beach band:

```
GroupHitbox([
    RectangleHitbox(right side),
    RectangleHitbox(top side),
    RectangleHitbox(left side),
    RectangleHitbox(bottom side)
])
```

`islandHitbox` is a single `RectangleHitbox(Vec(oceanSize, oceanSize), Vec(width-oceanSize, height-oceanSize))`.

### Step 5: River and trail generation

If `mapDef.rivers` or `mapDef.trails` exist, a `SeededRandom(this.seed)` is created and shared between both calls:

```typescript
const seededRandom = new SeededRandom(this.seed);
if (mapDef.trails) rivers.push(...this._generateRivers(mapDef.trails, seededRandom, true));
if (mapDef.rivers) rivers.push(...this._generateRivers(mapDef.rivers, seededRandom));
```

Trails are generated first so their seeded state is consumed before river generation.

**`_generateRivers` algorithm:**

1. Reads `RiverDefinition.{minAmount, maxAmount, wideChance, …}`.
2. Uses `seededRandom.getInt(minAmount, maxAmount)` for the count, then `mapScale(count)`.
3. Generates an array of widths: each entry is either `getInt(minWideWidth, maxWideWidth)` (if below `maxWideAmount` and `get() < wideChance`) or `getInt(minWidth, maxWidth)`.  The width array is sorted **descending** so wide rivers are placed first.
4. For each river, determines start position: if `centered`, the river starts at the map center edge; otherwise at a random point on the top or left edge (or bottom/right if `reverse`).
5. Calls `_generateRiver(start, startAngle, width, bounds, isTrail, rivers, centered, seededRandom)`.
6. A placement attempt is retried up to 100 times total.

**`_generateRiver` algorithm (point-walking with angular deviation):**

1. Starts from `startPos`, walks `60 points` (rivers) or `25 points` (trails).
2. At each step, an angular deviation is computed based on distance from map center:
   ```
   distFactor = distance(lastPoint, center) / (width / 2)
   maxDeviation = lerp(0.8, 0.1, distFactor)
   minDeviation = lerp(0.3, 0.1, distFactor)
   ```
   The farther from center, the smaller the allowed deviation — rivers tend to straighten near the edges.
3. Each point is offset from the previous by `Vec.fromPolar(angle, getInt(30, 80))`.
4. If the new point collides with a segment of an existing river/trail of the same type, the river terminates at the intersection (if `dist > 16` from the previous point).
5. If the point exits `bounds`, it is clamped and the walk ends.
6. Result is accepted only if `20 ≤ riverPoints.length ≤ 59`.
7. A `new River(width, riverPoints, existingRivers, mapBounds, isTrail)` is created and pushed.

### Step 6: Terrain construction

```typescript
this.terrain = new Terrain(width, height, mapDef.oceanSize, mapDef.beachSize, seed, rivers, waterType);
```

See [Terrain Class](#terrain-class) below. The `waterType` comes from the active mode's `replaceWaterBy` field (default `FloorNames.Water`).

### Step 7: Major building placement

```typescript
const majorBuildings = Array.from(mapDef.majorBuildings ?? []);
// optionally trim to maxMajorBuildings
majorBuildings.forEach(building => this._generateBuildings(building, 1));
```

Each call to `_generateBuildings` for a major building (one in `mapDef.majorBuildings`) enforces the quadrant constraint — one per quadrant, with a 150,000 unit distance² minimum between any two major buildings.

### Step 8: Regular building placement

```typescript
Object.entries(mapDef.buildings ?? {}).forEach(([building, count]) => this._generateBuildings(building, count));
```

For buildings without `bridgeHitbox`:
- Up to 400 random position attempts (`maxAttempts: 400` passed to `getRandomPosition`).
- `quadBuildingLimit` is enforced per-quadrant if the building has an entry.

For buildings with `bridgeHitbox` (bridges):
- Skips rivers that are trails or narrower than `bridgeMinRiverWidth`.
- Iterates fractional spline positions in `bridgeSpawnRanges` ranges (default `[[0.1, 0.4], [0.6, 0.9]]`, shuffled) to find the orientation closest to the river tangent direction.

### Step 9: Clearing generation

```typescript
this._generateClearings(mapDef.clearings);
```

A clearing is a `RectangleHitbox` with random dimensions between `minWidth×minHeight` and `maxWidth×maxHeight`. Up to 100 attempts are made to find a non-colliding position. Accepted clearings are stored in `this.clearings[]`.

Each clearing is then populated with obstacles: for each entry in `clearingDef.obstacles`, `random(obstacle.min, obstacle.max)` copies are spawned with `ignoreClearings: false` (inside the clearing). Obstacles in `mapDef.obstacles` whose `idString` is in `clearingDef.allowedObstacles` can also spawn inside clearings.

### Step 10: River and trail obstacle placement

```typescript
if (mapDef.rivers) this._generateRiverObstacles(mapDef.rivers, false);
if (mapDef.trails) this._generateRiverObstacles(mapDef.trails, true);
```

For each river, for each obstacle type in `RiverDefinition.obstacles`:

```typescript
const amount = riverDef.obstacles[obstacle] * river.width * river.points.length / 500;
```

This is a density formula — result is typically non-integer and represents an expected count. Each obstacle is placed at a random point inside the river/trail area using `river.getRandomPosition()`.

### Step 11: Obstacle clump generation

```typescript
for (const clump of mapDef.obstacleClumps ?? []) this._generateObstacleClumps(clump);
```

Each `ObstacleClump`:
1. Finds `mapScale(clumpDef.clumpAmount)` center positions.
2. For each center, picks `random(minAmount, maxAmount)` obstacles.
3. Places them in an arc: each obstacle is at angle `j * (τ / amount) + offset` and radial distance `radius`, plus jitter (`randomPointInsideCircle(position, jitter)`).

### Step 12: Standalone obstacle placement

```typescript
Object.entries(mapDef.obstacles ?? {}).forEach(([obstacle, count]) => this._generateObstacles(obstacle, count));
```

Count is scaled by `mapScale`. Position is found via `getRandomPosition` with the obstacle's `spawnMode` (typically `Grass` or `GrassAndSand`). Obstacles with `ignoreClearings: true` in their spawn config can spawn inside clearings.

### Step 13: Ground loot spawning

```typescript
Object.entries(mapDef.loots ?? {}).forEach(([loot, count]) => this._generateLoots(loot, count));
```

Each loot table entry generates `count` positions using `MapObjectSpawnMode.GrassAndSand` (beach-inclusive). For each position, `getLootFromTable(modeName, table)` returns an array of items, all added at the same position via `game.addLoot()` with `jitterSpawn: false`.

### Step 14: `onGenerate` hook

```typescript
mapDef.onGenerate?.(this, params);
```

Called after all automatic generation. `params` are the colon-separated segments from the original `mapData` string (beyond the map name). Map-specific plugins or spawn logic live here.

### Step 15: Named place positions

```typescript
if (mapDef.places) {
    packet.places = mapDef.places.map(({ name, position }) => ({
        name,
        position: Vec(
            this.width  * (position.x + randomFloat(-0.04, 0.04)),
            this.height * (position.y + randomFloat(-0.04, 0.04))
        )
    }));
}
```

Note: this step uses the **unseeded** `randomFloat`, so place label positions vary between games even with the same seed.

### Step 16: Serialization and buffer caching

```typescript
const stream = new PacketStream(new ArrayBuffer(1 << 16));
stream.serialize(packet);
this.buffer = stream.getBuffer();
```

A 64 KiB `ArrayBuffer` is used. The result is stored as `this.buffer` and served to every joining player without re-serialization.

## SeededRandom

Defined in `common/src/utils/random.ts`.

```typescript
export class SeededRandom {
    private _rng = 0;

    constructor(seed: number) {
        this._rng = seed;
    }

    /** Returns a float in [min, max). Default: [0, 1). */
    get(min = 0, max = 1): number {
        this._rng = this._rng * 16807 % 2147483647;
        return Numeric.lerp(min, max, this._rng / 2147483647);
    }

    /** Returns a rounded integer in [min, max]. Default: {0, 1}. */
    getInt(min = 0, max = 1): number {
        return Math.round(this.get(min, max));
    }
}
```

**Algorithm:** Park-Miller LCG (Lehmer RNG): `state = state * 16807 mod 2147483647`. The modulus is the Mersenne prime $2^{31} - 1$. The output is normalized by dividing by the modulus, then linearly interpolated into `[min, max)`.

**Why it matters for determinism:** The server sends `this.seed` in the `MapPacket`. The client constructs a `new Terrain(…, seed, rivers, …)`, which internally creates its own `new SeededRandom(seed)` to generate the jagged coast polygon. This means beach/grass polygon vertices are identical between server and client without the server transmitting the polygon geometry.

## Terrain Class

Defined in `common/src/utils/terrain.ts`.

```typescript
export class Terrain {
    readonly width: number;       // cells = Math.floor(mapWidth / 64)
    readonly height: number;      // cells = Math.floor(mapHeight / 64)
    readonly cellSize = 64;       // world units per grid cell

    readonly floors: Map<Hitbox, { floorType: FloorNames, layer: Layer | number }>;
    readonly rivers: readonly River[];
    readonly waterType: FloorNames;
    readonly beachHitbox: PolygonHitbox;
    readonly grassHitbox: PolygonHitbox;
    readonly groundRect: RectangleHitbox;
}
```

**Constructor** (`width, height, oceanSize, beachSize, seed, rivers, waterType`):

1. Allocates a `_grid[x][y]` of `{rivers[], floors[]}` for each 64-unit cell.
2. Generates the jagged coast with `jaggedRectangle(rect, spacing=16, variation=min(8, 2*oceanSize/beachSize), seededRandom)`. `jaggedRectangle` walks the four edges of the axis-aligned rectangle at 16-unit intervals and offsets each vertex by ±`variation/2`. This produces the polygon stored as `beachHitbox` and `grassHitbox`.
3. Populates river entries: for each `River`, computes the bounding rectangle of `bankHitbox`, rounds to cell indices, and adds the river to every cell whose 64×64 `RectangleHitbox` collides with `bankHitbox`.

**`addFloor(type, hitbox, layer)`** — registers an arbitrary floor hitbox (used by buildings for their interior floors). The hitbox is added to all overlapping grid cells for O(1) spatial lookup.

**`getFloor(position, layer)`** — resolves floor type at a world position:

1. Converts position to cell indices.
2. Default floor is `this.waterType` (ocean).
3. If inside `beachHitbox` polygon: on layer 0, result is `Grass` or `Sand` based on `grassHitbox`; on any other layer, result is `Void`.
4. Iterates rivers in the current cell: `bankHitbox` → Sand; `waterHitbox` → `this.waterType` (river). Breaks on first match.
5. Iterates registered floor hitboxes in the cell; returns the first matching floor on the correct layer.

**Why a 64-unit grid instead of direct polygon checks:** River and floor hitboxes are complex polygons (`PolygonHitbox`). Testing a point against every registered hitbox would be O(n) per query. The 64-unit cell grid reduces the candidate set to only those hitboxes intersecting the relevant cell, making `getFloor` nearly O(1) in practice.

**`getRiversInPosition(position)` and `getRiversInHitbox(hitbox)`** — cell-based queries for nearby rivers, used by obstacle placement and collision checks.

## River Class

Defined in `common/src/utils/terrain.ts`.

The `River` constructor takes the raw `points[]` (the walked path) and builds two `PolygonHitbox` shapes:

- `waterHitbox` (rivers only, not trails): the water channel polygon, width varies per point and widens near map edges via `(1 + end³ × 1.5) × width`.
- `bankHitbox`: the sand bank polygon, wider than water by `bankWidth = clamp(width × 0.75, 12, 20)`.

Bank/water polygons are built by projecting the perpendicular normal at each point outward by the appropriate width, with Catmull-Rom spline smoothing applied via:

```typescript
getPosition(t: number): Vector  // Catmull-Rom position along the spline
getTangent(t: number): Vector   // Catmull-Rom derivative (direction)
getNormal(t: number): Vector    // perpendicular to tangent
```

When two rivers' endpoints are within 48 units, the closer river's `bankHitbox` is used to clip the polygon edge so banks merge cleanly.

## Data Lineage

```
GameMap constructor receives (game, mapData, options)
  → split mapData → MapDefinition + params[]
  → compute mapScale function (identity or area-ratio scaler)
  → seed = random(0, 2^31)         [unseeded Math.random]
  → width = mapDef.width * scale
  → height = mapDef.height * scale
  → beachHitbox (GroupHitbox of 4 RectangleHitboxes)
  → islandHitbox (1 RectangleHitbox)
  → SeededRandom(seed) created for rivers only
  → _generateRivers(trails, seededRandom, isTrail=true)  [if trails]
  → _generateRivers(rivers, seededRandom, isTrail=false) [if rivers]
  → new Terrain(width, height, oceanSize, beachSize, seed, rivers, waterType)
      → jagged coast polygons generated via SeededRandom(seed) internally
      → bankHitboxes of rivers registered in 64-unit cell grid
  → majorBuildings: one per quadrant, dist² > 150,000
  → mapDef.buildings: regular buildings, per-quadrant limits enforced
  → _generateClearings: rectangular no-obstacle zones filled with obstacles
  → _generateRiverObstacles: density formula per river
  → _generateObstacleClumps: circular arc groups
  → _generateObstacles: standalone obstacles by spawn mode
  → _generateLoots: ground loot via MapObjectSpawnMode.GrassAndSand
  → mapDef.onGenerate?.(this, params)
  → place labels jittered and written to packet
  → PacketStream.serialize(packet) → this.buffer cached as ArrayBuffer
```

All placed objects are passed to `game.grid.addObject()` during their respective generation step.

## Complex Functions

### `GameMap` constructor — @file server/src/map.ts

**Purpose:** Runs the entire map generation pipeline in sequence.

**Implicit behavior:**
- Every placed `Building` and `Obstacle` is added to `game.grid` as it is created (during `generateBuilding` / `generateObstacle`), not in a batch at the end. This means later placement steps can query the grid and avoid collisions with earlier placements.
- `game.updateObjects = true` is set for each obstacle — picking up the initial dirty flag.
- `building_will_generate` / `obstacle_will_generate` plugin events are emitted before placement; returning `true` from either event vetoes generation of that object.
- `_packet.objects` is the array serialized into `MapPacket`. Only objects with `hideOnMap !== true` are pushed. Obstacles with `invisible: true` are also excluded.

**Called by:** `Game` constructor on game start.

### `Terrain` constructor — @file common/src/utils/terrain.ts

**Purpose:** Builds the jagged-polygon coast geometry and pre-indexes rivers into a 64-unit cellgrid.

**Implicit behavior:**
- Creates its own internal `new SeededRandom(seed)` — completely separate from the one used in `GameMap._generateRivers`. Both receive the same seed, so the jagged coast polygon is reproducible client-side from the seed alone.
- `beachHitbox` and `grassHitbox` use `PolygonHitbox`, not `RectangleHitbox` — point-in-polygon tests are costlier, but they are only called once per `getFloor` query and only if the position is near the coast.

**Called by:** `GameMap` constructor, passing the already-generated `River[]` array.

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Map Generation subsystem overview
- **Tier 2:** [../../spatial-grid/README.md](../../spatial-grid/README.md) — Grid that placed objects register into
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Map patterns
