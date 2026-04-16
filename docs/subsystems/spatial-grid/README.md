# Spatial Grid Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/spatial-grid/modules/ -->
<!-- @source: server/src/utils/grid.ts -->

## Purpose

The Grid is the server-side broad-phase spatial partitioning system. It divides the map
into fixed-size cells and tracks which game objects occupy which cells, allowing
collision detection and object-visibility queries to examine only the objects near a
given area instead of every object in the world.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/utils/grid.ts` | `Grid` class — spatial hash and object pool |
| `server/src/objects/gameObject.ts` | `BaseGameObject` — base class all stored objects inherit from |
| `common/src/utils/hitbox.ts` | `Hitbox` union type: `CircleHitbox`, `RectangleHitbox`, `GroupHitbox`, `PolygonHitbox` |
| `common/src/constants.ts` | `GameConstants.gridSize = 32` — canonical cell size constant |

## Architecture

The Grid uses a **2D array of lazily-initialised `Map` cells** (a spatial hash) together
with a reverse index so objects can be efficiently removed without scanning the whole grid.

### Internal Data Structures

```
_grid: Map<number, GameObject>[][]
  _grid[cellX][cellY] = Map<objectId, GameObject>   (created on first use)

_objectsCells: Map<number, Vector[]>
  _objectsCells.get(objectId) = [ {x, y}, {x, y}, … ]   (all cells this object occupies)

pool: ObjectPool<ObjectMapping>   (flat, non-spatial — all live objects by category)
```

- **`_grid`** — outer array (X axis) is pre-allocated at construction; inner arrays (Y axis)
  are plain JS arrays; individual cell `Map`s are created lazily with `??= new Map()`.
- **`_objectsCells`** — reverse map from `object.id` to the `Vector[]` of cells it was
  last registered in. Used by `_removeFromGrid` to delete the object without scanning
  every cell.
- **`pool`** — `ObjectPool<ObjectMapping>` that holds every live object regardless of
  spatial position, used for per-category iteration (e.g. all `Loot` objects each tick).

### Grid Cell Size

```
cellSize = 32   // readonly field on Grid
```

This matches `GameConstants.gridSize = 32` in `common/src/constants.ts`, though the class
does **not** import that constant — the value is hardcoded in the field declaration.

### Grid Dimensions

Constructed with the map's pixel width and height; dimensions are stored as cell counts:

```typescript
this.width  = Math.floor(width  / this.cellSize);   // pixel → cell
this.height = Math.floor(height / this.cellSize);
```

`MaxPosition = 1924` in `GameConstants` bounds the largest possible map, giving a maximum
of `Math.floor(1924 / 32) = 60` cells per axis.

## Data Flow

```
addObject(obj)
  └─ pool.add(obj)
  └─ updateObject(obj)          ← computes cells
       └─ _removeFromGrid(obj)  ← wipes previous cell entries
       └─ hitbox/spawnHitbox/position → toRectangle() → _roundToCells(min/max)
       └─ for each cell in [min..max]: _grid[x][y].set(obj.id, obj)
       └─ _objectsCells.set(obj.id, cells[])

intersectsHitbox(hitbox, layer?)
  └─ hitbox.toRectangle() → _roundToCells(min/max)
  └─ for each cell in [min..max]: collect objects from _grid[x][y]
  └─ (filter by adjacentOrEquivLayer if layer was supplied)
  └─ return Set<GameObject>

removeObject(obj)
  └─ _removeFromGrid(obj)       ← deletes from each cell in _objectsCells[obj.id]
  └─ pool.delete(obj)
```

Object movement (position change) is handled by callers invoking `grid.updateObject(this)`
after updating `_position`. The Grid does **not** track velocity or listen for events; each
object type is responsible for calling `updateObject` during its own tick.

## Key Methods

| Method | Parameters | Returns | Purpose |
|--------|-----------|---------|---------|
| `addObject` | `object: GameObject` | `void` | Registers object in pool and grid; warns on duplicate; sets `game.updateObjects = true` |
| `updateObject` | `object: GameObject` | `void` | Recomputes and re-registers the cells an object occupies (remove-then-add) |
| `removeObject` | `object: GameObject` | `void` | Removes object from all cells and from the pool |
| `intersectsHitbox` | `hitbox: Hitbox, layer?: Layer` | `Set<GameObject>` | Broad-phase query: returns all objects whose registered cells overlap the given hitbox bounds |

`pool` is a public `readonly` field (not a method); it exposes `pool.getCategory(ObjectCategory.X)`,
`pool.has(object)`, `pool.add(object)`, and `pool.delete(object)`.

### `updateObject` — Cell Assignment Logic

1. **No `hitbox` and no `spawnHitbox`** → object placed in a single cell at
   `_roundToCells(object.position)`.
2. **`spawnHitbox` present** (e.g., `Obstacle`) → `spawnHitbox.toRectangle()` used for bounds.
3. **`hitbox` present but no `spawnHitbox`** → `hitbox.toRectangle()` used for bounds.

In cases 2 and 3 every cell in the min→max bounding rectangle is registered.

### `_roundToCells` (private)

```typescript
private _roundToCells(vector: Vector): Vector {
    return {
        x: Numeric.clamp(Math.floor(vector.x / this.cellSize), 0, this.width),
        y: Numeric.clamp(Math.floor(vector.y / this.cellSize), 0, this.height)
    };
}
```

Converts world-space coordinates to cell indices. Out-of-bounds positions are clamped to
the grid edges, preventing array-out-of-bounds on objects near map borders.

## Layer Support

The Grid is **layer-aware at query time**, not at storage time. All objects are stored in
the same `_grid` cells regardless of layer. `intersectsHitbox` accepts an optional
`layer?: Layer`:

- **Omitted** — all objects in overlapping cells are returned, regardless of layer.
- **Provided** — objects are filtered by `adjacentOrEquivLayer(object, layer)` from
  `common/src/utils/layer.ts`, which passes objects on the same layer **or** an
  adjacent layer.

The `layer` field on `BaseGameObject` defaults to `Layer.Ground` and is mutable.

## Performance Characteristics

- **Insert/remove** — O(k) where k = number of cells the object's bounding rectangle
  covers. For most small objects k = 1–4.
- **Broad-phase query** — O(k + n) where k = cells in query rectangle and n = candidate
  objects returned. Returns a `Set` to avoid processing duplicates (an object spanning
  multiple cells would otherwise appear once per cell).
- **Per-category iteration** — O(objects in category) via `pool.getCategory()`; independent
  of spatial grid.
- **Memory** — cell `Map`s are allocated lazily; empty regions of the map cost only an
  `undefined` array slot.

## Interfaces & Contracts

Any object passed to `addObject` / `updateObject` / `removeObject` must be a `GameObject`
(the union type `ObjectMapping[ObjectCategory]`). `BaseGameObject` provides the required
fields:

| Field | Type | Used by Grid |
|-------|------|-------------|
| `id` | `number` | Key in cell `Map`s and `_objectsCells` |
| `position` | `Vector` | Fallback cell when no hitbox present |
| `hitbox` | `Hitbox \| undefined` | Bounding rectangle for cell registration |
| `spawnHitbox` | `Hitbox` (optional, contextual) | Overrides `hitbox` for grid placement when present |
| `layer` | `Layer` | Filtered in `intersectsHitbox` when a layer is provided |
| `type` | `ObjectCategory` | Used in pool category lookup and duplicate warning message |

## Dependencies

- **Depends on:**
  - [Object Definitions](../object-definitions/) — `common/src/utils/hitbox.ts` for
    `Hitbox` and `toRectangle()`; `common/src/constants.ts` for `Layer` enum
  - `common/src/utils/math.ts` — `Numeric.clamp` used in `_roundToCells`
  - `common/src/utils/layer.ts` — `adjacentOrEquivLayer` used in layer-filtered queries
- **Depended on by:**
  - [Game Loop](../game-loop/) — calls `addObject` / `removeObject` / `updateObject` when
    objects are created, destroyed, or move; calls `intersectsHitbox` for explosion radius,
    spawn-position checks, and airdrop placement; iterates `pool.getCategory()` each tick
  - [Map Generation](../map/) — reads grid indirectly through `Game` for placement
    collision checks during map load
  - All server-side objects that move (`Player`, `Loot`, `Projectile`, `SyncedParticle`,
    `Obstacle`) call `game.grid.updateObject(this)` from their own tick

## Module Index (Tier 3)

No separate Tier 3 modules exist yet. The spatial grid is monolithic; all logic is in [server/src/utils/grid.ts](../../../../../../server/src/utils/grid.ts).

## Known Gotchas

- **`spawnHitbox` takes priority over `hitbox`** — when `"spawnHitbox" in object` is true
  the regular hitbox is ignored for grid registration. This means queries using a regular
  hitbox may not find the object unless they overlap its spawn footprint.
- **No automatic movement tracking** — callers must invoke `grid.updateObject(obj)` after
  changing `obj.position` or the object's registered cells will be stale.
- **Bullets are NOT in the grid** — `Bullet` objects live in `game.bullets: Set<Bullet>`
  and are not `GameObject` instances; bullet–obstacle intersection is resolved separately.
- **Decals and DeathMarkers are static** — they are added to the grid but `updateObject`
  is never called for them after insertion.

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview
- **Tier 2:** [Game Loop](../game-loop/) — drives `addObject` / `removeObject` /
  `updateObject` calls and `pool.getCategory()` iteration each tick
- **Tier 2:** [Object Definitions](../object-definitions/) — defines hitbox types consumed
  by the grid
- **Patterns:** [patterns.md](patterns.md) — Grid usage patterns
