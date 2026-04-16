# Vector Operations & Collision Shapes

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: common/src/utils/vector.ts, common/src/utils/hitbox.ts, common/src/utils/math.ts -->

## Purpose

Low-level 2D vector arithmetic and collision detection shape implementations. Provides:
- **Immutable vector operations** — all methods return new Vector objects
- **Collision shape classes** — Circle, Rectangle, Polygon, and compound Group shapes
- **Ray-casting & intersection tests** — line-to-shape, shape-to-shape collision detection
- **Deterministic physics** — seeded RNG ensures identical results across server/client

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [common/src/utils/vector.ts](../../../../../common/src/utils/vector.ts#L1) | 2D vector interface and arithmetic (add, scale, normalize, dot product, projection) | Medium |
| [common/src/utils/hitbox.ts](../../../../../common/src/utils/hitbox.ts#L1) | Collision shape classes: `CircleHitbox`, `RectangleHitbox`, `PolygonHitbox`, `GroupHitbox` | High |
| [common/src/utils/math.ts](../../../../../common/src/utils/math.ts#L1) | Collision detection functions and intersection tests | High |

---

## Vector Structure

A 2D vector is represented by a simple interface with two number fields:

```typescript
interface Vector {
    x: number
    y: number
}
```

**Immutability:** All vector operations return **new Vector objects**. Original vectors are never modified.

### Standard Vector Construction

```typescript
// Both are equivalent:
const v = Vec(10, 20);
const v = { x: 10, y: 20 };
```

---

## Vector Operations

All operations follow the pattern: `Vec.operation(vector, params...)` returning a new Vector.

### Basic Arithmetic

| Function | Signature | Returns | Example |
|----------|-----------|---------|---------|
| `Vec(x, y)` | `(x: number, y: number) → Vector` | New vector with `x, y` | `Vec(5, 10)` |
| `Vec.add(a, b)` | `(a, b: Vector) → Vector` | Vector sum: `{x: a.x+b.x, y: a.y+b.y}` | `Vec.add(Vec(1,2), Vec(3,4))` → `{5, 6}` |
| `Vec.addComponent(a, x, y)` | `(a: Vector, x, y: number) → Vector` | `a` with scalar components added | `Vec.addComponent({2,3}, 1, 0)` → `{3, 3}` |
| `Vec.sub(a, b)` | `(a, b: Vector) → Vector` | Vector difference: `{x: a.x-b.x, y: a.y-b.y}` | `Vec.sub(Vec(5,5), Vec(2,1))` → `{3, 4}` |
| `Vec.subComponent(a, x, y)` | `(a: Vector, x, y: number) → Vector` | `a` with scalar components subtracted | `Vec.subComponent({5,5}, 2, 1)` → `{3, 4}` |
| `Vec.scale(a, n)` | `(a: Vector, n: number) → Vector` | Scalar multiplication: `{x: a.x*n, y: a.y*n}` | `Vec.scale({2,3}, 0.5)` → `{1, 1.5}` |
| `Vec.clone(v)` | `(v: Vector) → Vector` | Shallow copy of vector | `Vec.clone({x:1, y:2})` → `{x:1, y:2}` |
| `Vec.invert(a)` | `(a: Vector) → Vector` | Negation: `{x: -a.x, y: -a.y}` | `Vec.invert({1,2})` → `{-1, -2}` |

### Distance & Length

| Function | Signature | Returns | O() | Example |
|----------|-----------|---------|-----|---------|
| `Vec.squaredLen(a)` | `(a: Vector) → number` | Length squared: `a.x² + a.y²` | O(1) | `Vec.squaredLen({3,4})` → `25` |
| `Vec.len(a)` | `(a: Vector) → number` | Euclidean length: `√(a.x² + a.y²)` | O(1) | `Vec.len({3,4})` → `5` |

**Performance note:** Prefer `squaredLen()` for comparisons (avoids sqrt). Use `len()` only when absolute distance needed.

### Direction & Rotation

| Function | Signature | Returns | Example |
|----------|-----------|---------|---------|
| `Vec.direction(a)` | `(a: Vector) → number` | Angle in radians via `atan2(a.y, a.x)` | `Vec.direction({1,0})` → `0` |
| `Vec.rotate(a, angle)` | `(a: Vector, angle: number) → Vector` | Rotate by radians: `{x: a.x·cos - a.y·sin, y: a.x·sin + a.y·cos}` | `Vec.rotate({1,0}, π/2)` → `{0, 1}` |
| `Vec.fromPolar(angle, magnitude)` | `(angle: number, magnitude: number) → Vector` | Convert polar (angle, magnitude) to Cartesian | `Vec.fromPolar(0, 5)` → `{5, 0}` |

### Normalization

| Function | Signature | Returns | Notes |
|----------|-----------|---------|-------|
| `Vec.normalize(a)` | `(a: Vector) → Vector` | Unit vector parallel to `a`, or `{1, 0}` if `a` is too small | Returns original if `\|a\| < 1e-6` |
| `Vec.normalizeSafe(a, fallback)` | `(a: Vector, fallback?: Vector) → Vector` | Unit vector or provided `fallback` (default: `{1, 0}`) | Always safe; never throws |

**Gotcha:** Zero vectors normalize to fallback (default `{1, 0}`) to avoid NaN. Verify your input magnitude if unexpected direction occurs.

### Projection & Dot Product

| Function | Signature | Returns | O() | Example |
|----------|-----------|---------|-----|---------|
| `Vec.dotProduct(a, b)` | `(a, b: Vector) → number` | Scalar: `a.x·b.x + a.y·b.y` | O(1) | `Vec.dotProduct({1,0}, {0,1})` → `0` |
| `Vec.project(projected, projectOnto)` | `(projected, projectOnto: Vector) → Vector` | Vector projection of `projected` onto `projectOnto` | O(1) | Project `{2,2}` onto `{1,0}` → `{2,0}` |
| `Vec.angleBetweenVectors(a, b)` | `(a, b: Vector) → number` | Angle between vectors via `acos(dotProduct / (len·len))` | O(1) | Angle between `{1,0}` and `{0,1}` → `π/2` |

### Interpolation & Comparison

| Function | Signature | Returns | Example |
|----------|-----------|---------|---------|
| `Vec.lerp(start, end, t)` | `(start, end: Vector, t: number) → Vector` | Linear interpolation: `start·(1-t) + end·t` | `Vec.lerp({0,0}, {10,10}, 0.5)` → `{5, 5}` |
| `Vec.equals(a, b, epsilon)` | `(a, b: Vector, epsilon: number) → boolean` | Approximate equality within tolerance | Tolerance default: `0.001` |
| `Vec.addAdjust(pos1, pos2, orientation)` | `(pos1, pos2: Vector, orientation: Orientation) → Vector` | Add `pos2` rotated by orientation (0-3) to `pos1` | Used for rotatable objects |

---

## Collision Shapes

Collision shapes form a type hierarchy with a common abstract base:

```
BaseHitbox (abstract)
├─ CircleHitbox
├─ RectangleHitbox
├─ PolygonHitbox (partial support)
└─ GroupHitbox
```

All shapes implement:
- `collidesWith(other: Hitbox): boolean` — Fast boolean test
- `getIntersection(other: Hitbox): CollisionResponse` — Collision normal & penetration depth
- `resolveCollision(other: Hitbox): void` — Move this shape to resolve intersection
- `distanceTo(other: Hitbox): CollisionRecord` — Distance and collision status
- `intersectsLine(start, end: Vector): IntersectionResponse` — Ray-casting
- `isPointInside(point: Vector): boolean` — Point containment
- `randomPoint(): Vector` — Random point within bounds
- `transform(position, scale?, orientation?): Hitbox` — Transform and return new hitbox

### CircleHitbox

Circle defined by **center position** and **radius**.

```typescript
class CircleHitbox extends BaseHitbox {
    position: Vector  // Center point
    radius: number    // Distance from center
    
    constructor(radius: number, position?: Vector)
}
```

**Constructor shorthands:**
- `CircleHitbox.simple(radius, center?)` → JSON export format

**Collision implementation:**
- Circle ↔ Circle: Distance-based (sum of radii)
- Circle ↔ Rectangle: Clamp circle center to rect bounds, measure distance to edge
- Circle ↔ Group: Test each child shape
- Circle ↔ Polygon: Falls back to rectangle approximation (TODO)

**Special methods:**
| Function | Returns |
|----------|---------|
| `toRectangle()` | Bounding rectangle: `min={pos.x-r, pos.y-r}`, `max={pos.x+r, pos.y+r}` |
| `randomPointInsideCircle(center, radius)` | Point uniformly distributed in disk |

### RectangleHitbox

Rectangle defined by **min and max corners** (axis-aligned).

```typescript
class RectangleHitbox extends BaseHitbox {
    min: Vector  // Top-left (lower x, lower y bounds)
    max: Vector  // Bottom-right (higher x, higher y bounds)
    
    constructor(min: Vector, max: Vector)
}
```

**Constructor shorthands:**
- `RectangleHitbox.fromRect(width, height, center?)` → Create from dimensions
- `RectangleHitbox.fromLine(a, b)` → Create from two endpoints (bounding rect)
- `RectangleHitbox.simple(width, height, center?)` → JSON export format

**Collision implementation:**
- Rect ↔ Rect: AABB overlap test
- Rect ↔ Circle: Clamp circle center to bounds, distance to edge
- Rect ↔ Group: Test each child
- Rect ↔ Polygon: Line intersection tests

**Key property:**
```typescript
isFullyWithin(other: RectangleHitbox): boolean
  // true if this rect is completely inside other rect
```

**Special method:**
```typescript
getSide(point: Vector): Orientation | -1
  // Returns which side of rectangle the point is on
  // 0 = top, 1 = right, 2 = bottom, 3 = left, -1 = corner
```

### PolygonHitbox

Polygon with a list of **ordered vertices** and center point.

```typescript
class PolygonHitbox extends BaseHitbox {
    points: Vector[]  // Ordered vertices (n ≥ 3)
    center: Vector    // Polygon center/origin
    
    constructor(points: Vector[], center?: Vector)
}
```

**Constructor shorthands:**
- `PolygonHitbox.simple(points, center?)` → JSON export format

**Collision implementation:**
- Polygon ↔ Rectangle: Vertex containment + edge intersection tests
- Polygon ↔ Circle: Falls back to rect approximation (not implemented)
- Polygon ↔ Group: Test each child
- Polygon ↔ Polygon: Not implemented (would need SAT)

**Current limitations:**
- ⚠️ **`resolveCollision()` throws error** — use only for detection
- ⚠️ **`distanceTo()` throws error** — use only for detection
- ⚠️ **`intersectsLine()` throws error** — use `collidesWith()` instead
- ✅ **`isPointInside()` works** — ray-casting algorithm (even-odd rule)

**Point containment algorithm:**
```typescript
isPointInside(point: Vector): boolean
  // Ray-casting: shoot ray from point, count boundary crossings
  // Even count = outside, odd count = inside
```

### GroupHitbox

Compound shape containing **multiple child shapes** (Circle or Rectangle).

```typescript
class GroupHitbox extends BaseHitbox {
    hitboxes: (CircleHitbox | RectangleHitbox)[]  // Child shapes
    
    constructor(...hitboxes: (CircleHitbox | RectangleHitbox)[])
}
```

**Constructor shorthands:**
- `GroupHitbox.simple(...hitboxes)` → JSON export format

**How it works:**
- `collidesWith()` → **true** if **any child** collides
- `getIntersection()` → Returns collision with **deepest penetration** among all children
- `resolveCollision()` → Delegates to child shapes
- `distanceTo()` → Returns **closest distance** among all children
- `intersectsLine()` → Returns **closest** line intersection

**Use cases:**
- Multi-part objects (e.g., building with multiple rooms)
- Complex hitbox layouts (e.g., L-shaped obstacle)
- Performance optimization (group related shapes)

---

## Raycasting & Line Intersection

Ray-casting tests determine if a line segment intersects a shape and returns collision details.

### Line-Line Intersection

```typescript
// Returns intersection point, or null if parallel/non-intersecting
Collision.lineIntersectsLine(startA, endA, startB, endB): Vector | null
  // Test two infinite lines

Collision.lineSegmentIntersection(startA, endA, startB, endB): Vector | null
  // Test two line SEGMENTS (bounded by endpoints)
  // Uses parametric form with Cramer's rule
```

**Example:**
```typescript
const hit = Collision.lineSegmentIntersection(
  {x: 0, y: 0},     // Ray start
  {x: 10, y: 0},    // Ray end
  {x: 5, y: -5},    // Segment start
  {x: 5, y: 5}      // Segment end
);
// Returns {x: 5, y: 0} if within bounds, null otherwise
```

### Line-Circle Intersection

```typescript
Collision.lineIntersectsCircle(
  s0: Vector,       // Ray start
  s1: Vector,       // Ray end
  pos: Vector,      // Circle center
  rad: number       // Circle radius
): IntersectionResponse  // {point, normal} or null
```

**Returns:**
- `null` if no intersection
- `{point, normal}` where:
  - `point` = intersection position on ray
  - `normal` = unit vector pointing outward from circle center

**Example:**
```typescript
const hit = Collision.lineIntersectsCircle(
  {x: 0, y: 0},    // Ray origin
  {x: 10, y: 0},   // Ray end
  {x: 5, y: 0},    // Circle center
  3                // Radius 3
);
// Returns {point: {x: 2, y: 0}, normal: {x: -1, y: 0}}
```

### Line-Rectangle Intersection

```typescript
Collision.lineIntersectsRect(
  s0: Vector,       // Ray start
  s1: Vector,       // Ray end
  min: Vector,      // Rectangle min corner
  max: Vector       // Rectangle max corner
): IntersectionResponse  // {point, normal} or null
```

**Algorithm:** DDA (Digital Differential Analyzer)
- Traces ray parameter `t` from 0 to 1
- Records entering/exiting intersections with each axis-aligned slab
- Returns **first hit point** and surface normal

**Returns:**
- `point` = intersection on ray
- `normal` = surface normal `(±1, 0)` or `(0, ±1)`

**Test-only variant (boolean return):**
```typescript
Collision.lineIntersectsRectTest(s0, s1, min, max): boolean
```

---

## Shape-to-Shape Collision

### Collision Response Format

Collision functions return either **null** (no collision) or a response object:

```typescript
type CollisionResponse = {
    readonly dir: Vector      // Direction to push shape (unit vector)
    readonly pen: number      // Penetration depth (units to separate)
} | null

type CollisionRecord = {
    readonly collided: boolean
    readonly distance: number  // Squared distance if not colliding
}
```

**Resolve collision example:**
```typescript
const collision = Collision.circleCircleIntersection(
  circleA.position, circleA.radius,
  circleB.position, circleB.radius
);

if (collision) {
  // Move circle A away from B by penetration depth
  circleA.position = Vec.sub(
    circleA.position,
    Vec.scale(collision.dir, collision.pen)
  );
}
```

### Circle-Circle Collision

```typescript
// Boolean test (fast)
Collision.circleCollision(
  centerA: Vector, radiusA: number,
  centerB: Vector, radiusB: number
): boolean

// Detailed response
Collision.circleCircleIntersection(
  centerA: Vector, radiusA: number,
  centerB: Vector, radiusB: number
): CollisionResponse

// Distance
Collision.distanceBetweenCircles(...): CollisionRecord
```

**Algorithm:** 
```
distance = √((a.x - b.x)² + (a.y - b.y)²)
collided = distance < (radiusA + radiusB)
```

**O() Complexity:** O(1)

### Circle-Rectangle Collision

```typescript
Collision.rectangleCollision(
  min: Vector, max: Vector,    // Rectangle bounds
  pos: Vector, rad: number     // Circle center & radius
): boolean

Collision.rectCircleIntersection(
  min: Vector, max: Vector,
  pos: Vector, rad: number
): CollisionResponse

Collision.distanceBetweenRectangleCircle(
  min: Vector, max: Vector,
  circlePos: Vector, circleRad: number
): CollisionRecord
```

**Algorithm:**
1. Clamp circle center to rectangle bounds → closest point on rect
2. If distance from point to center < radius → collision
3. If circle center inside rect → special handling (push along longest axis)

**O() Complexity:** O(1)

### Rectangle-Rectangle Collision (AABB)

```typescript
Collision.rectRectCollision(
  min1: Vector, max1: Vector,
  min2: Vector, max2: Vector
): boolean

Collision.rectRectIntersection(
  min0: Vector, max0: Vector,
  min1: Vector, max1: Vector
): CollisionResponse

Collision.distanceBetweenRectangles(...): CollisionRecord
```

**Algorithm (AABB overlap):**
```
// Ranges overlap if: minA < maxB AND minB < maxA
overlap_x = min2.x < max1.x AND min1.x < max2.x
overlap_y = min2.y < max1.y AND min1.y < max2.y
collided = overlap_x AND overlap_y
```

**Penetration direction:** Along axis with least overlap (choose x or y)

**O() Complexity:** O(1)

---

## Rotation & Angles

Angle operations handle radians, normalization, and orientation conversions.

### Angle Constants

```typescript
const π = Math.PI;              // 3.14159...
const halfπ = π / 2;            // π/2, 90°
const τ = 2 * π;                // 2π, 360°

// Accessibility aliases
const PI = π;
const HALF_PI = halfπ;
const TAU = τ;
```

### Angle Normalization

```typescript
Angle.normalize(radians: number): number
  // Clamp angle to [-π, π]
  // Example: 3π → -π, π + 0.1 → -π + 0.1

Angle.minimize(start: number, end: number): number
  // Shortest rotation from start → end
  // Example: from 10° to 350° returns -20° (not 340°)
```

**Why normalize?** Prevents angle wraparound issues:
```typescript
// Without normalize:
angle_diff = 350° - 10° = 340°   // Wrong! Should be -20°

// With normalize:
angle_diff = Angle.minimize(10°, 350°) = -20°  // Correct!
```

### Angle Between Points

```typescript
Angle.betweenPoints(a: Vector, b: Vector): number
  // Angle of line from a → b
  // Uses atan2(a.y - b.y, a.x - b.x)
```

### Angle Conversion

```typescript
Angle.degreesToRadians(degrees: number): number
  // 360° → 2π, 180° → π, etc.
  // Formula: deg * π / 180

Angle.radiansToDegrees(radians: number): number
  // π → 180°, 2π → 360°, etc.
  // Formula: rad / π * 180

Angle.orientationToRotation(orientation: 0-3): number
  // Convert discrete orientation (0=up, 1=right, 2=down, 3=left)
  // to rotation angle for PixiJS sprite display
  // Formula: -normalize(orientation * π/2)
```

### Range Testing

```typescript
Angle.isAngleInside(
  angle: number,
  start: number,
  end: number
): boolean
  // True if angle is within arc from start → end (clockwise)
  // Handles wrap-around (e.g., 350° → 10°)
```

---

## Hitbox Compound Operations

### Hitbox Serialization

All hitboxes can be serialized to/from JSON:

```typescript
interface HitboxJSON {
  type: HitboxType.Circle | Rect | Group | Polygon
  // ... type-specific fields
}

// Serialize
const json: HitboxJSON = hitbox.toJSON();

// Deserialize
const hitbox = BaseHitbox.fromJSON(json);
```

### Hitbox Transformation

Transform a hitbox (translate, scale, rotate) and get a **new** hitbox:

```typescript
hitbox.transform(
  position: Vector,              // Translation
  scale?: number,                // Scale factor (default: 1)
  orientation?: Orientation      // Rotation: 0=0°, 1=90°, 2=180°, 3=270°
): Hitbox
```

**Note:** Polygons rotate by 90° intervals only (via orientation, not radians).

### Hitbox Scaling

Scale a hitbox **in-place**:

```typescript
hitbox.scale(scale: number): void
  // MultiplesAll dimensions by scale
  // For rects: scales from center
  // For circles: radius *= scale
```

### Hitbox Bounds

Get axis-aligned bounding rectangle (always works, even for polygons):

```typescript
hitbox.toRectangle(): RectangleHitbox
  // Returns `min` and `max` corners encompassing entire shape
```

---

## Performance Notes

### Complexity Analysis

| Operation | Shape | Complexity | Notes |
|-----------|-------|-----------|-------|
| Circle-Circle collision | Circle × Circle | O(1) | Just distance formula |
| Circle-Rect collision | Circle × Rect | O(1) | Clamp + distance |
| Rect-Rect collision | Rect × Rect | O(1) | AABB overlap test |
| Point-in-Circle | Point ↔ Circle | O(1) | Single distance check |
| Point-in-Rect | Point ↔ Rect | O(1) | Four comparisons |
| Point-in-Polygon | Point ↔ Polygon | O(n) | Ray-casting, n = vertex count |
| Line-Circle intersection | Line ↔ Circle | O(1) | Quadratic formula |
| Line-Rect intersection | Line ↔ Rect | O(1) | DDA algorithm |
| Group collision | Group × Shape | O(m) | m = number of children |

### Optimization Tips

1. **Use `sqaredLen()`** for distance comparisons (avoid sqrt)
   ```typescript
   // ❌ Slow: involves sqrt
   if (Vec.len(diff) < threshold) { }
   
   // ✅ Fast: squared comparison
   if (Vec.squaredLen(diff) < threshold * threshold) { }
   ```

2. **Broad-phase first** — use AABB before expensive collision tests
   ```typescript
   const rect = circle.toRectangle();  // Get bounds
   if (!circle.collidesWith(rect)) return null;  // Early exit
   // ... then detailed test
   ```

3. **Cache intersection functions** — spatial grid stores hitbox type pairs
   ```typescript
   const isRegistered = intersectionFunctions[typeA][typeB];
   // Avoid registration lookups per collision check
   ```

4. **Use GroupHitbox wisely** — adds iteration overhead
   ```typescript
   // ❌ Many small circles in scene = many collision checks per frame
   objects.forEach(obj => obj.hitbox.collidesWith(other));
   
   // ✅ Group related shapes = fewer high-level checks
   const grouped = new GroupHitbox(circle1, circle2, circle3);
   ```

---

## Gotchas & Numerical Precision

### Zero Vector Normalization

Normalizing a zero vector returns a fallback:
```typescript
Vec.normalize({x: 0, y: 0})      // Returns {x: 1, y: 0}
Vec.normalizeSafe({x: 0, y: 0}) // Returns fallback or {x: 1, y: 0}
```

**Why?** NaN is poisonous — permeates all downstream calculations. Safer to use default direction.

**Mitigation:** Check vector magnitude before normalizing if you need to detect zero vectors:
```typescript
const mag = Vec.len(dir);
if (mag < 0.001) {
  // Handle zero vector specially
} else {
  direction = Vec.normalize(dir);
}
```

### Angle Wrap-Around

Angles are cyclic — `π` and `-π` are the same direction:
```typescript
2π + 0.1  →  0.1   (wraps around after 2π)
-π - 0.1  →  π - 0.1  (wraps around before -π)
```

**Always normalize** after angle arithmetic:
```typescript
let angle = 2π + 0.5;
angle = Angle.normalize(angle);  // Now: 0.5
```

### Orientation vs Radians

Game uses two angle systems:

| Type | Range | Use | Example |
|------|-------|-----|---------|
| **Orientation** | 0-3 (4 directions) | Discrete rotations (buildings, props) | 0=up, 1=right, 2=down, 3=left |
| **Radians** | -π to π (continuous) | Smooth rotations (players, projectiles) | 0=right, π/2=down |

**Convert carefully:**
```typescript
// ❌ Wrong: orientation is not an angle
sprite.rotation = orientation;  // Will display incorrectly

// ✅ Correct: use Angle.orientationToRotation()
sprite.rotation = Angle.orientationToRotation(orientation);
// Returns: -0, -π/2, -π, -3π/2 (depending on orientation)
```

### Polygon Limitations

Current polygon hitbox has **incomplete support**:
- ✅ Collision detection with rectangles
- ✅ Point containment test
- ❌ Collision with circles (falls back to bounding rect)
- ❌ Line intersection (throws error)
- ❌ Collision resolution for physics

**Use polygons for detection-only** — check collisions but don't resolve.

### Distance to Line Edge Case

When finding distance from point to line segment, endpoints are clamped:
```typescript
// Point beyond segment end → distance = distance to endpoint
Collision.distanceToLine(
  {x: 10, y: 0},    // Point far beyond segment
  {x: 0, y: 0},     // Segment start
  {x: 1, y: 0}      // Segment end
);
// Returns: distance from point to {x: 1, y: 0} (endpoint)
// Not: perpendicular distance to infinite line
```

---

## Related Documentation

- **Tier 2:** See [Core Math & Physics README](../README.md) for subsystem architecture
- **Tier 2:** [Serialization System](../serialization-system/README.md) — Binary encoding of vectors in packets
- **Tier 2:** [Spatial Grid](../spatial-grid/README.md) — Uses hitbox bounds for broad-phase collision
- **Tier 2:** [Projectiles & Ballistics](../projectiles-ballistics/README.md) — Uses line-circle intersection for bullet hit detection
- **Tier 1:** [Data Model](../../datamodel.md) — How vectors/hitboxes relate to game objects
- **Tier 1:** [Architecture](../../architecture.md) — Physics determinism requirement
