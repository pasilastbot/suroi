# Spatial Queries Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/utils/grid.ts -->

## Purpose

Implements spatial hashing grid for efficient broad-phase collision detection and proximity queries, enabling O(1) region lookups instead of O(n) linear scans.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/utils/grid.ts` | `Grid` class, spatial hash, region queries | High |
| `common/src/utils/hitbox.ts` | Hitbox-to-rectangle transformation for region bounds | High |
| `server/src/game.ts` | Grid instantiation and object lifecycle | Medium |

## Business Rules

- **Fixed Cell Size:** Grid divides map into 32×32 unit cells (defined by `Grid.cellSize = 32`)
- **Dynamic Grid Layout:** Grid cells are created on-demand (lazily) to save memory
- **Layered Queries:** Hitbox queries can filter by layer (Layer.Ground, Layer.Basement, etc.) or return all layers
- **Object Position Tracking:** Each object maintains list of cells it occupies (for fast removal on move)
- **Hitbox as AABB:** Hitbox.toRectangle() converts any rotation/shape to axis-aligned bounding box for grid queries
- **No Early Removal:** Objects must be explicitly removed via `removeObject()`; no automatic timeout

## Data Lineage

```
Object Movement (position change)
    ↓
Game loop calls grid.updateObject(object)
    ↓
Grid.updateObject():
  - Remove from old cells (via _objectsCells map)
  - Compute new cell range from hitbox rectangle
  - Add to all intersecting cells
    ↓
Grid._roundToCells() — converts world coordinates to grid (x,y) indices
    ↓
_grid[x][y] stores Map<id, GameObject>
    ↓
intersectsHitbox(hitbox, layer) — returns all objects in touched cells (broad-phase)
```

## Dependencies

- **Internal:** `Hitbox` (for AABB computation), `ObjectPool` (tracks all objects)
- **External:** Game object types (all inherit `BaseGameObject`)
- **Used by:** Bullet raycast, proximity checks, explosion radius queries

## Grid Structure

| Component | Purpose |
|-----------|---------|
| `_grid: Map<number, GameObject>[][]` | 2D array of cell maps (sparse — created on-demand) |
| `_objectsCells: Map<number, Vector[]>` | Caches which grid cells each object occupies (for fast removal) |
| `cellSize: 32` | Fixed pixels per cell |
| `width`, `height` | Grid dimensions in cells (world size ÷ cellSize) |

## Complex Functions

### `Grid.constructor(game: Game, width: number, height: number)`  
**Purpose:** Initialize spatial grid for a map.

**Implicit Behavior:**
- Divides map dimensions by `cellSize` (32) to get grid width/height
- Creates empty 2D array (_grid)
- Initializes object pool and cells tracking map

**Gotcha:** If map is 1920×1920, grid is 60×60 cells. Cell (0,0) is world (0,0); cell (60,60) is world (1920,1920).

**File:** `server/src/utils/grid.ts:20–32`

### `Grid.updateObject(object: GameObject): void`  
**Purpose:** Recompute which grid cells an object occupies after movement.

**Implicit Behavior:**
1. Remove object from previous cells (via `_removeFromGrid`)
2. Compute hitbox or position
3. If object has no hitbox: place in single cell at position
4. If object has hitbox: compute AABB, round to cells, add to all intersecting cells
5. Store cell list in `_objectsCells[object.id]`
6. Mark game as needing updates

**Performance:** O(k) where k = cells touched by hitbox (typically < 16 for small objects, < 100 for buildings)

**Gotcha:** Does NOT update game tick budget. Many updatesper frame → cumulative cost.

**File:** `server/src/utils/grid.ts:57–99`

### `Grid.intersectsHitbox(hitbox: Hitbox, layer?: Layer): Set<GameObject>`  
**Purpose:** Broad-phase query—get all objects touching a region.

**Implicit Behavior:**
1. Convert hitbox to AABB rectangle
2. Round bounds to grid cells
3. Iterate all cells in range
4. Collect objects from each cell
5. If layer specified: filter by `adjacentOrEquivLayer(object, layer)`
6. Return Set of matching objects

**Performance:** O(c + m) where c = cells in range, m = objects in those cells. Typical: 4–16 objects returned for 1-cell query.

**Layer Logic:** Two layers are compatible if they're the same or adjacent (e.g., Ground and ToUpstairs are adjacent; Basement and Ground are not).

**File:** `server/src/utils/grid.ts:145–175`

### `Grid._roundToCells(position: Vector): Vector`  
**Purpose:** Convert world coordinates to grid cell indices.

**Implicit Behavior:**
- Divides x, y by cellSize (32), floors result
- Returns cell (x, y) index

**Example:** Position (64.5, 128.3) → cell (2, 4)

**File:** `server/src/utils/grid.ts:224–226`

## Scale Considerations

| Map Size | Grid Cells | Typical Object Count | Query Cost |
|----------|-----------|---------------------|------------|
| 1280×1280 (debug) | 40×40 (1,600 cells) | 0–50 | < 1 ms |
| 1920×1920 (normal) | 60×60 (3,600 cells) | 50–200 | 2–5 ms |
| 2560×2560 (large) | 80×80 (6,400 cells) | 200–500 | 5–15 ms |

**Bottleneck:** Not grid queries, but collision detection after broad-phase (narrow-phase hitbox intersections).

## Related Documents

- **Tier 2:** [Server Utilities Subsystem](../README.md) — Utility overview
- **Tier 2:** [Spatial Grid Subsystem](../../spatial-grid/README.md) — Grid design and alternatives
- **Tier 1:** [Core Math & Physics](../../core-math-physics/README.md) — AABB, vector math, hitbox
- **Patterns:** [Spatial Grid Patterns](../patterns.md) — Query patterns and optimization
