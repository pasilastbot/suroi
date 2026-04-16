# Collision & Hitbox System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: common/src/utils/hitbox.ts -->

## Purpose

Provides collision detection framework for the game. The hitbox system defines shape hierarchies (Circle, Rectangle, Polygon, Group) for representing object boundaries, collision testing (overlap, distance, raycasting), and collision resolution for physics and game logic. Used throughout the engine for:
- Physics and obstacle collision
- Attack range & hit detection
- Projectile trajectory tracking
- Object culling and spatial queries
- Visual boundary testing

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [common/src/utils/hitbox.ts](../../../common/src/utils/hitbox.ts) | Hitbox class hierarchy (Circle, Rectangle, Polygon, Group) and shape implementations |
| [common/src/utils/math.ts](../../../common/src/utils/math.ts) | Collision detection algorithms (circle-circle, circle-rect, rect-rect, line intersection) |
| [server/src/utils/grid.ts](../../../server/src/utils/grid.ts) | Spatial partitioning for collision queries (delegates to hitbox methods) |
| [server/src/objects/gameObject.ts](../../../server/src/objects/gameObject.ts) | Base object model with hitbox property |

## Shape Hierarchy

The hitbox system is built on a type-dispatched design pattern. All shapes extend `BaseHitbox` abstract class:

```
BaseHitbox (abstract)
  ├─ CircleHitbox       | Fast; O(1) distance formula
  ├─ RectangleHitbox    | AABB; common for buildings/obstacles
  ├─ GroupHitbox        | Union of shapes; can contain Circles & Rectangles
  └─ PolygonHitbox      | Precise; O(vertex) ray-casting algorithms
```

**Type dispatch:** Intersection functions are registered in a lookup table `intersectionFunctions[typeA][typeB]`, allowing the runtime to select the appropriate collision algorithm without pattern matching.

### Shape-Specific Characteristics

| Shape | Use Cases | Performance | Notes |
|-------|-----------|-------------|-------|
| **Circle** | Player, projectiles, explosion radius | O(1) | Stored as `position` + `radius` |
| **Rectangle** | Buildings, obstacles, AABB bounding boxes | O(1) | Stored as `{min, max}` points (AABB) |
| **Group** | Complex obstacles (e.g., building with multiple rooms) | O(n children) | Union of Circles or Rectangles; iterates all children |
| **Polygon** | Precise water/boundary shapes, map features | O(v) per test | Stored as vertex array; uses ray-casting |

---

## Collision Operations

### 1. **Collision Detection** — `collidesWith(that: Hitbox): boolean`

Tests overlap without detailed collision info. Returns `true` if shapes intersect.

**Per shape:**

| Test | Algorithm |
|------|-----------|
| **Circle-vs-Circle** | Distance formula: `dist(center1, center2) < radius1 + radius2` |
| **Circle-vs-Rect** | Closest point on rectangle to circle center; if distance < radius, collides |
| **Rect-vs-Rect** | AABB overlap: `min1.x < max2.x && min1.y < max2.y && min2.x < max1.x && min2.y < max1.y` |
| **Polygon-vs-Rect** | Separating Axis Theorem (SAT); check if any polygon edge separates shapes |
| **Group-vs-any** | Iterate children; return `true` if ANY child collides |

**Performance implications:**
- Circle-Circle: fastest (2 square roots)
- Rect-Rect: very fast (4 comparisons)
- Polygon: slowest (O(vertices) tests per edge)

### 2. **Intersection Details** — `getIntersection(that: Hitbox): CollisionResponse`

Returns detailed collision response: `{ dir: Vector, pen: number }` (direction & penetration depth).

- **dir:** Normalized direction to push `this` to resolve collision
- **pen:** Distance to separate shapes (negative = penetration depth)

Used for physics resolution, attack knockback calculation.

Example: If circle overlaps rectangle by 5 units, response is `{ dir: [1, 0], pen: 5 }` (push right by 5 units).

### 3. **Distance Measurement** — `distanceTo(that: Hitbox): CollisionRecord`

Returns `{ collided: boolean, distance: number }` (squared distance, or negative if overlapping).

Used for:
- Detecting if objects are within interaction range (e.g., item pickup)
- Sorting nearby objects by proximity
- Spatial queries ("all objects within 100 units")

### 4. **Collision Resolution** — `resolveCollision(that: Hitbox): void`

Mutates `this` hitbox position to separate from collision.

- **Circle:** Moves center along collision direction
- **Rectangle:** Translates min/max bounds
- **Group:** Delegates to each child
- **Polygon:** Not implemented (throws error)

Used by physics engine to prevent objects from overlapping.

### 5. **Line Intersection** — `intersectsLine(a: Vector, b: Vector): IntersectionResponse`

Returns `{ point: Vector, normal: Vector }` where line segment `a → b` first intersects the hitbox.

**Per shape:**

| Shape | Algorithm |
|-------|-----------|
| **Circle** | Parametric line-circle intersection (solve quadratic) |
| **Rect** | Test intersections with all 4 edges; return closest point |
| **Polygon** | Iterate edges; test edge-to-line segment via parametric solver |
| **Group** | Collect all intersections; return closest to line start |

**Use cases:**
- Raycasting for projectiles (bullet trajectory)
- Line-of-sight tests (can player see target?)
- Melee attack cone (check if enemies in sweep arc)

---

## Point-in-Shape Testing

### `isPointInside(point: Vector): boolean`

Tests if a point lies inside the hitbox.

**Per shape:**

| Shape | Algorithm |
|-------|-----------|
| **Circle** | `distance(point, center) < radius` |
| **Rect** | AABB containment: `point.x > min.x && point.x < max.x && point.y > min.y && point.y < max.y` |
| **Polygon** | Ray-casting: cast ray from point; count edge crossings; odd = inside |
| **Group** | Return `true` if ANY child contains point |

Used for:
- Checking if projectile hit target
- Player interaction detection (click on object)
- Inventory item pickup ("is player close enough?")

### `isFullyWithin(that: RectangleHitbox): boolean` (RectangleHitbox only)

Tests if this rectangle is completely inside another rectangle (`that`).

Used for visibility culling, spawn point validation.

---

## Hitbox Transformations

### `transform(position: Vector, scale?, orientation?): Hitbox`

Returns a **new hitbox** (non-mutating) with applied transformations.

**Applied in order:**
1. **Position:** Offset hitbox center by `position` vector
2. **Scale:** Multiply radius/dimensions by `scale` factor
3. **Orientation:** Rotate around center by `orientation` (0-3 for cardinal directions)

Example:
```typescript
const original = new CircleHitbox(5, Vec(0, 0));
const rotated = original.transform(Vec(10, 10), 1.5, 2);
// Result: Circle at (10, 10) with radius 7.5, rotated 180°
```

**For rectangles:** Uses `Geometry.transformRectangle()` which applies rotation to bounds.

**For polygons:** Applies transformation to each vertex individually.

### `scale(scale: number): void`

Mutates hitbox in-place by scaling (used for **live-updated** objects).

- **Circle:** Multiplies radius
- **Rect:** Scales bounds around center point
- **Polygon:** Scales each vertex around origin

---

## Bounds & Centers

### `toRectangle(): RectangleHitbox`

Converts any shape to its axis-aligned bounding box (AABB).

- **Circle:** AABB with width/height = 2×radius
- **Rect:** Returns clone
- **Polygon:** Finds min/max x, y across all vertices
- **Group:** Finds union of all child bounds

**Critical for grid system:** Grid stores objects in cells based on `toRectangle()` bounds.

### `getCenter(): Vector`

Returns geometric center of hitbox.

- **Circle:** Center position
- **Rect:** Midpoint of bounds
- **Polygon:** Center of bounding rectangle (not centroid)
- **Group:** Center of union bounds

---

## Spatial Grid Integration

The [Spatial Grid](../spatial-grid/) uses hitboxes for efficient collision queries.

### Grid Storage

**Grid cells:** 32×32 pixels (constant `GameConstants.gridSize`)

**Object storage:** When object's hitbox changes, grid:
1. Converts hitbox to rectangle via `toRectangle()`
2. Rounds rectangle bounds to grid cells
3. Adds object ID to ALL intersecting cells
4. Stores cell list for fast removal

**Query:** `grid.intersectsHitbox(hitbox, layer?)` returns all objects near the hitbox in O(grid cells touched) time.

### Data Flow

```
GameObject.hitbox (set)
  ↓
Grid.updateObject(object)
  ├─ Convert hitbox → rectangle
  ├─ Round to grid cells
  └─ Store object ID in all cells
  ↓
Later: Grid.intersectsHitbox(test_hitbox)
  ├─ Find cells this hitbox touches
  ├─ Collect objects from those cells
  ├─ Filter by layer (optional)
  └─ Return Set<GameObject>
```

---

## Serialization

### `toJSON() / fromJSON()`

Converts hitbox to/from JSON for **networking and persistence**.

**JSON format:**

```json
{
  "type": 0,
  "radius": 5,
  "position": { "x": 10, "y": 20 }
}
```

```json
{
  "type": 1,
  "min": { "x": 0, "y": 0 },
  "max": { "x": 30, "y": 40 }
}
```

Used for:
- Sending object definitions to clients
- Loading map geometry from JSON data files
- Testing/debugging (serialize hitbox state)

---

## Performance Characteristics

### Collision Testing Complexity

| Test | Time | Space | Notes |
|------|------|-------|-------|
| **Circle-Circle** | O(1) | O(1) | Two distance checks |
| **Circle-Rect** | O(1) | O(1) | Clamping + distance |
| **Rect-Rect** | O(1) | O(1) | Four comparisons (AABB) |
| **Polygon-Rect** | O(v) | O(1) | v = polygon vertices |
| **Group-Any** | O(n) | O(1) | n = child shapes |
| **Line-Shape** | O(v) | O(1) | v = shape complexity |

### Spatial Grid Reduces O(n²)

**Without grid:** Checking collision for 100 objects = 100 × 99 / 2 = 4,950 tests

**With 32px grid in 640×480 map:** ~20 grid cells touched per object; average ~5 objects per cell = ~25 checks per object = 2,500 total

### Vector Operations

Use squared distances where possible to avoid `Math.sqrt()`:
- `distanceSquared(a, b)` → `(a.x - b.x)² + (a.y - b.y)²`
- Compare `distSquared < radiusSquared` instead of `distance < radius`

---

## Common Gotchas & Known Issues

### 1. **GroupHitbox contains Array of Shapes**

```typescript
new GroupHitbox(circle1, circle2, rect1)  // ✓ OK
new GroupHitbox(group1)                   // ✗ No nesting
```

GroupHitbox cannot contain other GroupHitbox. Flatten before adding.

### 2. **PolygonHitbox Limited Collision**

```typescript
polygon.collidesWith(circle)  // ✓ Works (via toRectangle)
polygon.resolveCollision(x)   // ✗ Throws error
polygon.distanceTo(x)         // ✗ Throws error
```

Polygon hitboxes are **read-only**; only support collision detection and line intersection. For resolution, use `toRectangle()` fallback.

### 3. **Grid Cell Size Fixed**

```typescript
const gridSize = 32;  // Hard-coded in grid.ts
```

Objects **much larger than 32px** touch many cells (inefficient). Objects **much smaller** share cells with many other objects. No optimization for non-uniform object sizes.

### 4. **Hitbox Mutation in Transform**

```typescript
const h1 = new CircleHitbox(5, Vec(0, 0));
const h2 = h1.transform(Vec(10, 10)); // ✓ h1 unchanged
h1.scale(2);                          // ✓ h1 mutated in-place
```

`transform()` is non-mutating; `scale()` is mutating. Be careful when chaining.

### 5. **Rectangle Intersection Asymmetrical (GroupHitbox)**

```typescript
const group = new GroupHitbox(circle, rect);
group.getIntersection(other)              // Returns deepest collision
// But no guarantee which child's collision is returned
```

GroupHitbox returns collision with deepest penetration, but if two children have equal penetration, behavior is undefined.

### 6. **Line Intersection Returns Closest Point**

```typescript
intersectsLine(start, end)  // Returns intersection CLOSEST to start
```

If line passes through multiple shapes, only returns the first intersection. No list of all intersections.

### 7. **Polygon Ray-Casting Not Optimized**

Polygon collision detection iterates all vertices without optimization. Large/complex polygons (>20 vertices) can be slow.

### 8. **Orientation Encoding**

```typescript
orientation: 0  // North (↑)
orientation: 1  // East  (→)
orientation: 2  // South (↓)
orientation: 3  // West  (←)
```

Must use 0-3 encoding; no radian support in hitbox transforms.

---

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Component map, grid placement
- [Data Model](../../datamodel.md) — Hitbox type definitions, object schema

### Tier 2 — Subsystems
- [Spatial Grid](../spatial-grid/) — Grid spatial partitioning for collision queries
- [Game Objects (Server)](../game-objects-server/) — Objects with hitboxes
- [Projectiles & Ballistics](../projectiles-ballistics/) — Bullet raycasting via hitbox
- [Core Math & Physics](../core-math-physics/) — Vector operations used by hitbox

### Tier 3 — Modules
- (None yet — hitbox system is core, not subdivided into modules)

### Patterns
- (See patterns.md for common hitbox usage patterns)

