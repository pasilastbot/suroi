# Query & Optimization

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/utils/grid.ts -->

## Purpose

Documents the spatial grid implementation for O(1) object lookup and broad-phase collision culling. Explains grid dimensions, query algorithms, and performance characteristics.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/utils/grid.ts` | Grid class, cell-based lookup, range queries | High |
| `server/src/game.ts` | Grid updates on object movement, collision queries | Medium |
| `common/src/constants.ts` | Grid cell size (32), map bounds (1924) | Low |

## Grid Implementation

### Dimensions & Cell Size

```typescript
// server/src/utils/grid.ts
export class Grid {
    readonly width: number;      // Width in cells
    readonly height: number;     // Height in cells
    readonly cellSize = 32;      // Pixels per cell (from GameConstants)
    
    // 2D array: Map<objectId, GameObject> per cell
    private readonly _grid: Map<number, GameObject>[][];
    
    constructor(game: Game, mapWidth: number, mapHeight: number) {
        // Map bounds in pixels: [0, maxPosition] × [0, maxPosition]
        // maxPosition = 1924 (from common/src/constants.ts)
        
        this.width = Math.floor(mapWidth / this.cellSize);
        this.height = Math.floor(mapHeight / this.cellSize);
        
        // Grid dimensions: ceil(1924 / 32) = 61 cells × 61 cells
        // But typically maps are 32×32 to 100×100 cells
        
        // Initialize grid with empty arrays (cells created on-demand)
        this._grid = Array.from({ length: this.width + 1 }, () => []);
    }
}
```

**Spatial bounds:**

```
Map: [0, 1924] × [0, 1924] pixels
Grid: [0, 61] × [0, 61] cells (1924 / 32 ≈ 60.125)
Cell size: 32 pixels × 32 pixels
```

**Typical map (32×32 cells):**

```
Map: [0, 1024] × [0, 1024] pixels
Grid: [0, 32] × [0, 32] cells
Cell size: 32 pixels
```

### Grid Cell Organization

Each cell is a `Map<objectId, GameObject>` for O(1) ID lookup:

```typescript
private readonly _grid: Map<number, GameObject>[][];

// Example: Grid[10][15] contains all objects occupying cell (10, 15)
const cellMap: Map<number, GameObject> = this._grid[10][15];

// Lookup object in cell:
const obj = cellMap.get(objectId);  // O(1) average case

// Iterate objects in cell:
for (const obj of cellMap.values()) {
    // Process each object
}
```

### Object-to-Cell Mapping

When an object is added/moved, the grid tracks which cells it occupies:

```typescript
// server/src/utils/grid.ts
private readonly _objectsCells = new Map<number, Vector[]>();

// Example:
// _objectsCells.set(42, [
//     { x: 10, y: 15 },  // Object 42 occupies cell (10, 15)
//     { x: 10, y: 16 },  // and cell (10, 16)
//     { x: 11, y: 15 },  // and cell (11, 15)
//     { x: 11, y: 16 }   // and cell (11, 16)
// ]);

// Fast removal: O(cells occupied), not O(grid size)
removeObject(object): void {
    const cells = this._objectsCells.get(object.id);
    for (const cell of cells) {
        this._grid[cell.x][cell.y].delete(object.id);
    }
}
```

## Cell-Based Object Lookup: O(1)

### Adding an Object

```typescript
// server/src/utils/grid.ts
addObject(object: GameObject): void {
    this.pool.add(object);
    this.updateObject(object);  // Place in grid
}

updateObject(object: GameObject): void {
    this._removeFromGrid(object);  // Remove from old cells
    const cells: Vector[] = [];
    
    // Determine cells this object occupies
    const hitbox = object.hitbox || { toRectangle() { /* point */ } };
    const rect = hitbox.toRectangle();
    
    const min = this._roundToCells(rect.min);
    const max = this._roundToCells(rect.max);
    
    // Add object to all cells in bounding box
    for (let x = min.x; x <= max.x; x++) {
        for (let y = min.y; y <= max.y; y++) {
            (this._grid[x][y] ??= new Map()).set(object.id, object);
            cells.push(Vec(x, y));
        }
    }
    
    this._objectsCells.set(object.id, cells);
}

private _roundToCells(position: Vector): Vector {
    return {
        x: Numeric.clamp(Math.floor(position.x / this.cellSize), 0, this.width),
        y: Numeric.clamp(Math.floor(position.y / this.cellSize), 0, this.height)
    };
}
```

**Time complexity:** O(cells occupied) — typically 1-4 cells per object, so O(1) amortized.

## Range Queries

### Radius Query (Circular Range)

Find all objects within distance R of a point:

```typescript
// server/src/utils/grid.ts
intersectsHitbox(hitbox: Hitbox, layer?: Layer): Set<GameObject> {
    const rect = hitbox.toRectangle();
    
    const min = this._roundToCells(rect.min);
    const max = this._roundToCells(rect.max);
    
    const objects = new Set<GameObject>();
    const includeAll = layer === undefined;
    
    // Iterate all cells in hitbox's bounding box
    for (let x = min.x; x <= max.x; x++) {
        for (let y = min.y; y <= max.y; y++) {
            const cellMap = this._grid[x][y];
            if (!cellMap) continue;  // Cell is empty
            
            // Check each object in cell against actual hitbox
            for (const obj of cellMap.values()) {
                if (includeAll || (obj.layer && adjacentOrEquivLayer(obj, layer))) {
                    // Caller will do precise hitbox-to-hitbox collision
                    objects.add(obj);
                }
            }
        }
    }
    
    return objects;
}
```

**Example:** Explosion with blast radius 100 pixels

```
Explosion center: (500, 500)
Blast radius: 100 pixels
Hitbox: CircleHitbox(500, 500, 100)
Bounding box in cells:
    min = (500 - 100) / 32 = 12.5 → 12
    max = (500 + 100) / 32 = 18.75 → 18
    
Cells to query: [12, 18] × [12, 18] = 49 cells (worst case)
Objects found: Typically 5-15 (checked against precise distance)

Time: O(49 cells × avg 2 objects/cell) = O(100) ≪ O(10000 total objects)
```

### Rectangle Query

Find all objects in axis-aligned rectangular region:

```typescript
queryRectangle(minX: number, minY: number, maxX: number, maxY: number): Set<GameObject> {
    const min = this._roundToCells({ x: minX, y: minY });
    const max = this._roundToCells({ x: maxX, y: maxY });
    
    const objects = new Set<GameObject>();
    
    for (let x = min.x; x <= max.x; x++) {
        for (let y = min.y; y <= max.y; y++) {
            const cellMap = this._grid[x][y];
            if (cellMap) {
                for (const obj of cellMap.values()) objects.add(obj);
            }
        }
    }
    
    return objects;
}
```

## Broad-Phase Culling in Collision Detection

Game loop uses grid to avoid checking every object pair (O(n²) → O(n)):

```typescript
// server/src/game.ts
tick(): void {
    // ... update positions ...
    
    // Collision detection with culling
    for (const bullet of this.bullets) {
        // Find candidates using spatial grid (O(log n))
        const hitCandidates = this.grid.intersectsHitbox(
            bullet.hitbox,
            bullet.layer
        );
        
        // Check precise collisions only against candidates
        for (const candidate of hitCandidates) {
            if (collides(bullet, candidate)) {
                // Handle collision
                bullet.hit(candidate);
            }
        }
        
        // Without culling, would check against ALL 1000+ objects
        // With culling, check only 10-20 candidates
    }
}
```

**Performance improvement:**

```
Without grid: 1000 bullets × 10000 objects = 10,000,000 checks/tick
With grid: 1000 bullets × 15 candidates = 15,000 checks/tick
Speedup: ~666×
```

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `addObject(obj)` | O(c) | c = cells occupied (usually 1-4) |
| `removeObject(obj)` | O(c) | Remove from c cells |
| `updateObject(obj)` | O(c) | Remove + re-add |
| `intersectsHitbox(hitbox)` | O(c + m) | c = cells in bounding box, m = objects in those cells |
| `queryRectangle(x1,y1,x2,y2)` | O(c + m) | Same as above |

### Memory Complexity

```
Grid size: (width + 1) × (height + 1) array
        = ~(1924/32 + 1)^2 = ~65^2 = ~4225 cells

Per-object cells tracking: n objects × avg 2 cells = ~2n storage
Total grid memory: ~4225 + 2n = ~O(grid_size + n)

For typical 100-player game:
= 4225 + (100 + obstacles + loot + bullets) × 2
= 4225 + ~1000 = ~5.2 KB
```

### Spatial Locality Benefits

Grid enables:

1. **Culling for rendering:** Only render visible objects → 60 FPS on client
2. **Culling for sound:** Only play sounds from nearby objects → lower CPU
3. **Culling for damage:** Gas damage only checks nearby players
4. **Area-of-effect queries:** Grenade explosion blast finds affected players in ~O(log n)

## Grid Update Requirements

Objects must call `grid.updateObject()` when position changes by more than cell size:

```typescript
// server/src/objects/player.ts
set position(pos: Vector) {
    this._position = pos;
    this.game.grid.updateObject(this);  // ← CRITICAL: updates cell occupancy
}
```

**Gotcha:** If `grid.updateObject()` is not called after position change:

```
Old grid cells: (10, 15), (10, 16), (11, 15), (11, 16)
Position moved to: (50, 50)  // Now in cells (25, 25)
Grid still thinks object is at (10, 15)

Result: intersectsHitbox() won't find this object
        Collision detection misses it
        Other objects phase through it
```

## Spatial Grid Metrics & Profiling

```typescript
// server/src/game.ts
getGridStats(): {
    cellCount: number,
    objectCount: number,
    avgObjectsPerCell: number,
    maxObjectsInCell: number,
    emptyCells: number
} {
    let totalObjects = 0;
    let maxInCell = 0;
    let emptyCellCount = 0;
    
    for (let x = 0; x <= this.grid.width; x++) {
        for (let y = 0; y <= this.grid.height; y++) {
            const cell = this._grid[x][y];
            if (!cell || cell.size === 0) {
                emptyCellCount++;
            } else {
                totalObjects += cell.size;
                maxInCell = Math.max(maxInCell, cell.size);
            }
        }
    }
    
    return {
        cellCount: (this.grid.width + 1) * (this.grid.height + 1),
        objectCount: this.grid.pool.size,
        avgObjectsPerCell: totalObjects / (this.grid.cellCount - emptyCellCount),
        maxObjectsInCell: maxInCell,
        emptyCells: emptyCellCount
    };
}
```

**Healthy grid:** ~60% empty cells, avg 0.5-2 objects/occupied cell, max 10-20 in busiest cell.

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Spatial grid subsystem overview
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — System architecture
- **Tier 3:** [../../game-objects-server/modules/object-lifecycle.md](../../game-objects-server/modules/object-lifecycle.md) — Object registration in grid
- **Collision:** [../../collision-hitbox/README.md](../../collision-hitbox/README.md) — Hitbox types and collision logic
- **Game Loop:** [../../game-loop/README.md](../../game-loop/README.md) — Collision phase in tick
