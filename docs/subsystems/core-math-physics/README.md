# Core Math & Physics Utilities

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/core-math-physics/modules/ -->
<!-- @source: common/src/utils/ -->

## Purpose

Foundation library providing deterministic physics, collision detection, and geometry calculations used by both server and client. All calculations are **seeded and deterministic** — same inputs always yield identical results, a critical requirement for network synchronization in a multiplayer game.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [common/src/utils/math.ts](../../../../common/src/utils/math.ts) | Angle math, trigonometry, geometry, collision detection, interpolation |
| [common/src/utils/vector.ts](../../../../common/src/utils/vector.ts) | 2D vector interface and arithmetic operations |
| [common/src/utils/hitbox.ts](../../../../common/src/utils/hitbox.ts) | Collision shape classes (Circle, Rectangle, Polygon, Group) |
| [common/src/utils/baseBullet.ts](../../../../common/src/utils/baseBullet.ts) | Shared bullet ballistics configuration and effects |
| [common/src/utils/layer.ts](../../../../common/src/utils/layer.ts) | Z-layer system (basement, ground, upstairs) and layer predicates |
| [common/src/utils/random.ts](../../../../common/src/utils/random.ts) | `SeededRandom` deterministic RNG for reproducible gameplay |

## Architecture

All physics is **deterministic via seeding** — identical random seeds guarantee identical simulation results:

```
Server: rand = new SeededRandom(0x12345678)
  │
  ├─ Generate projectile trajectory (position, velocity, angle)
  ├─ Calculate collision hits (ray-casts via hitbox.intersectsLine())
  └─ Calculate damage falloff via distance
  
  → Serialize delta in UpdatePacket

Client: Receives UpdatePacket
  │
  ├─ Deserialize projectile delta (server's calculated state)
  ├─ Render projectile visually (no recalculation — trust server)
  └─ Apply effects (tracer, trail particles)
```

**Key insight:** The client **never** re-runs physics simulations. It receives authoritative deltas from the server and renders them. This prevents cheating and keeps simulation logic server-side.

## Vector Operations

Vectors are **immutable** — all operations return new Vector objects. The `Vec` object provides both construction (`Vec(x, y)`) and operation methods.

### Vector API

| Operation | Function | Parameters | Returns | Use Case |
|-----------|----------|-----------|---------|----------|
| Create | `Vec(x, y)` | `x: number, y: number` | `Vector` | Construct vector |
| Add | `Vec.add(a, b)` | vectors | `Vector` | Position + velocity |
| Subtract | `Vec.sub(a, b)` | vectors | `Vector` | Direction calculation |
| Scale | `Vec.scale(v, n)` | vector, scalar | `Vector` | Velocity scaling |
| Rotate | `Vec.rotate(v, angle)` | vector, radians | `Vector` | Projectile trajectory after bounce |
| Length (squared) | `Vec.len(a)` / `Vec.squaredLen(a)` | vector | `number` | Distance/range checks |
| Direction | `Vec.direction(v)` | vector | `number` (radians) | Aim angle |
| Normalize | `Vec.normalize(v)` | vector | `Vector` | Unit direction vector |
| Dot product | `Vec.dotProduct(a, b)` | vectors | `number` | Perpendicularity test |
| Project | `Vec.project(v, onto)` | vectors | `Vector` | Surface-normal reflection |
| Clone | `Vec.clone(v)` | vector | `Vector` | Create copy (immutability pattern) |
| Component add | `Vec.addComponent(a, x, y)` | vector, x, y | `Vector` | Position adjustments |

### Vector Type Signature

```typescript
interface Vector {
    x: number
    y: number
}

// Vec acts as both constructor and namespace
const pos = Vec(100, 200);                    // Create vector
const moved = Vec.add(pos, Vec(10, 5));       // Add immutably
const distance = Vec.len(Vec.sub(a, b));      // Chain operations
```

## Hitbox System

Collision detection uses a **shape hierarchy**. Each shape type has optimized intersection logic.

### Hitbox Type Hierarchy

```
BaseHitbox (abstract)
├─ CircleHitbox              [fastest; used for damage radius, projectiles]
├─ RectangleHitbox           [common; used for buildings, obstacles, players]
├─ PolygonHitbox             [precise, slower; used for complex obstacles]
└─ GroupHitbox               [union of multiple shapes; e.g., building with rooms]
```

### HitboxType Enum

```typescript
enum HitboxType {
    Circle = 0,     // Fastest intersection test (~O(1) distance calculation)
    Rect = 1,       // Common for axis-aligned bounds (AABB)
    Group = 2,      // Union of shapes
    Polygon = 3     // Vertices; precise but slower
}
```

### Hitbox Core Methods

| Method | Returns | Purpose | When Used |
|--------|---------|---------|-----------|
| `collidesWith(other)` | `boolean` | AABB overlap test | Broad phase; fast reject |
| `resolveCollision(other)` | `void` | Separate overlapping shapes | Physics resolution; push back |
| `distanceTo(other)` | `CollisionRecord` | Distance & collision status | Damage range, trigger radius |
| `getIntersection(other)` | `CollisionResponse` | Collision normal & penetration | Resolve multi-shape groups |
| `intersectsLine(a, b)` | `IntersectionResponse` | Ray-cast hit point & normal | Hitscan weapons, line-of-sight |
| `randomPoint()` | `Vector` | Random point inside shape | Loot spawn, particle effects |
| `isPointInside(p)` | `boolean` | Point containment test | Trigger check |
| `transform(pos, scale, orientation)` | `Hitbox` | Transform copy | Rotate/scale/move objects |
| `clone(deep)` | `Hitbox` | Create copy (deep or shallow) | Object allocation |
| `toRectangle()` | `RectangleHitbox` | AABB approximation | Simplification for fallback |

### Collision Response Types

```typescript
// Result of intersection test
interface CollisionRecord {
    readonly collided: boolean   // Shapes intersect?
    readonly distance: number    // Negative if collided; how much to separate
}

// Ray-cast result
type IntersectionResponse = {
    readonly point: Vector       // Where line intersects shape
    readonly normal: Vector      // Surface normal at intersection
} | null;                         // null if no intersection

// Collision resolution (pushback vector)
type CollisionResponse = {
    readonly dir: Vector         // Direction to separate
    readonly pen: number         // Penetration depth (how much to push)
} | null;
```

### Example: Hitbox Creation

```typescript
// From common/src/definitions/obstacles.ts pattern

// Circle hitbox (fast)
const damageRadius = new CircleHitbox(32, {x: player.x, y: player.y});

// Rectangle hitbox (building)
const buildingBounds = new RectangleHitbox(
    {x: -50, y: -50},   // min
    {x: 50, y: 50}      // max
);

// Group hitbox (building with multiple rooms)
const buildingGroup = new GroupHitbox(
    new RectangleHitbox(...),  // main structure
    new RectangleHitbox(...)   // interior room
);

// Check collision
if (damageRadius.collidesWith(buildingBounds)) {
    // Resolve pushback
    damageRadius.resolveCollision(buildingBounds);
}

// Ray-cast (hitscan weapon)
const hit = buildingBounds.intersectsLine(gunPos, targetPos);
if (hit) {
    console.log(`Hit at ${hit.point}, surface normal ${hit.normal}`);
}
```

## Layer System

Layers represent **vertical stratification** in the game world — ground level, basements, upper floors. Each layer has distinct rendering order and collision visibility.

### Layer Enum

```typescript
enum Layer {
    Basement = -2,      // Underground area
    ToBasement = -1,    // Staircase down (transition)
    Ground = 0,         // Main level (default)
    ToUpstairs = 1,     // Staircase up (transition)
    Upstairs = 2        // Upper floor
}

// Collision behavior mode
enum Layers {
    All = 0,            // Collide with objects on all layers (rare)
    Adjacent = 1,       // Collide same layer + adjacent layers (default)
    Equal = 2           // Only collide with objects on same layer
}
```

### Layer Predicates (common/src/utils/layer.ts)

| Function | Purpose | Example |
|----------|---------|---------|
| `isGroundLayer(layer)` | Returns `true` if layer is `Ground` or `Basement` | Check if object is on solid surface |
| `isStairLayer(layer)` | Returns `true` if layer is transition (`ToBasement`, `ToUpstairs`) | Handle stair physics |
| `equalLayer(a, b)` | Exact layer match | Same-level collision only |
| `adjacentOrEqualLayer(ref, eval)` | Match same layer or one level above/below | Player can see stairs above |
| `equivLayer(refObj, evalObj)` | Match considering object definitions | Some obstacles block all layers |

### Layer Rendering & Collision Rules

```
Basement (-2)
  ↓
Staircase down (-1) ←→ Staircase up (+1)
  ↓                      ↑
Ground (0) ←─ Main walkable area (default spawn)
               Collision visible from both Ground and adjacent Basement/Upstairs
  ↓
Staircase up (+1) ←→ Staircase down (-1)
  ↓
Upstairs (+2)
```

**Collision visibility:**
- `Layers.All` — object collides across all layers (e.g., large explosion)
- `Layers.Adjacent` — object collides with same layer + ±1 layer (default for most objects)
- `Layers.Equal` — object collides only on same layer (walls don't cross layers)

### Switching Layers

Typically done via stair obstacles with `isStair: true` definition. Walking on a stair transitions the player from one layer to the next.

## Random Number Generation

Deterministic seeded RNG ensures **reproducible gameplay**. The `SeededRandom` class uses a linear congruential generator (LCG) — not cryptographically secure, but sufficient for game simulation.

### SeededRandom API

```typescript
class SeededRandom {
    constructor(seed: number)

    /**
     * Generate random float
     * @param [min = 0] minimum value (included)
     * @param [max = 1] maximum value (excluded)
     * @returns random float in [min, max)
     */
    get(min?: number, max?: number): number

    /**
     * Generate random integer
     * @param [min = 0] minimum (included)
     * @param [max = 1] maximum (included)
     * @returns random integer in [min, max]
     */
    getInt(min?: number, max?: number): number
}
```

### Determinism Guarantee

```typescript
const seed = 0x12345678;

// Server simulation
const server_rng = new SeededRandom(seed);
const server_pos = randomPointInsideCircle({x: 100, y: 100}, 32); // depends on seed

// Client replay (same seed → same result)
const client_rng = new SeededRandom(seed);
const client_pos = randomPointInsideCircle({x: 100, y: 100}, 32); // identical to server_pos

// Both produce exact same position — crucial for network sync
```

### Global Random Utilities (Unseeded — Browser Math.random)

For **visual-only** effects that don't need determinism:

| Function | Purpose |
|----------|---------|
| `randomFloat(min, max)` | Random float (uses `Math.random()`) |
| `random(min, max)` | Random integer (uses `Math.random()`) |
| `randomBoolean()` | Random true/false (uses `Math.random()`) |
| `randomSign()` | Random ±1 (uses `Math.random()`) |
| `randomVector(minX, maxX, minY, maxY)` | Random 2D vector (uses `Math.random()`) |
| `randomRotation()` | Random angle (uses `Math.random()`) |
| `randomPointInsideCircle(center, radius, [minRadius])` | Random point in circle (uses `Math.random()`) |
| `weightedRandom(items, weights)` | Pick from weighted list (uses `Math.random()`) |
| `pickRandomInArray(items)` | Pick random element (uses `Math.random()`) |

**Important:** Unseeded functions use browser `Math.random()` and are **NOT deterministic**. Use `SeededRandom` for anything affecting gameplay.

## Math Utilities

### Angle Math (Angle namespace)

All angles in **radians**. Conversion utilities provided.

| Function | Purpose | Returns | Example |
|----------|---------|---------|---------|
| `Angle.betweenPoints(a, b)` | Vector direction from a→b | radians | Aim angle |
| `Angle.normalize(rad)` | Normalize to [-π, π] | radians | Canonical angle representation |
| `Angle.minimize(start, end)` | Shortest angle difference | radians | Rotation delta |
| `Angle.degreesToRadians(degrees)` | Convert | radians | API compatibility |
| `Angle.radiansToDegrees(radians)` | Convert | degrees | Debug output |
| `Angle.orientationToRotation(orientation)` | Orientation→radians | radians | Rotation from tile orientation |
| `Angle.isAngleInside(angle, start, end)` | Angle in arc? | boolean | Cone check (melee/shotgun) |

### Numeric Utilities (Numeric namespace)

| Function | Purpose | Returns | Example |
|----------|---------|---------|---------|
| `Numeric.absMod(a, n)` | Modulo handling negatives | number | Cycle layers: `absMod(layer-1, layers)` |
| `Numeric.lerp(start, end, factor)` | Linear interpolation | number | Smooth movement: `lerp(pos, targetPos, 0.5)` |
| `Numeric.delerp(t, a, b)` | Inverse lerp (parameter) | number | Normalize value to [0, 1] |
| `Numeric.clamp(value, min, max)` | Range limit | number | Bound velocity, health |
| `Numeric.addOrientations(n1, n2)` | Orientation arithmetic | Orientation | Rotation composition |
| `Numeric.remap(value, min0, max0, min1, max1)` | Range remap | number | Damage scaling per distance |
| `Numeric.min(a, b)` / `Numeric.max(a, b)` | Min/max (generic) | number \| bigint | Bounds |
| `Numeric.equals(a, b, epsilon)` | Float equality (tolerance) | boolean | 3D→2D depth cull |

### Geometry Utilities (Geometry namespace)

| Function | Purpose | Returns | Example |
|----------|---------|---------|---------|
| `Geometry.distance(a, b)` | Euclidean distance | number | Range check |
| `Geometry.distanceSquared(a, b)` | Distance² (faster; avoids sqrt) | number | Optimization: compare squared distances |
| `Geometry.signedArea(a, b, c)` | Signed area of triangle | number | Winding number (point-in-polygon) |
| `Geometry.transformRectangle(pos, min, max, scale, orientation)` | Transform bounds | `{min, max}` | Rotated rectangle bounding box |

### Collision Utilities (Collision namespace)

Exact collision detection routines. Scene queries typically use hitbox methods instead.

| Function | Purpose | Returns | Example |
|----------|---------|---------|---------|
| `Collision.circleCollision(centerA, radiusA, centerB, radiusB)` | Circle-circle overlap | `boolean` | Damage radius check |
| `Collision.rectangleCollision(min, max, pos, rad)` | Rectangle-circle overlap | `boolean` | Player in room? |
| `Collision.rectRectCollision(min1, max1, min2, max2)` | Rectangle-rectangle overlap | `boolean` | AABB broad phase |
| `Collision.distanceBetweenCircles(...)` | Circle-circle distance & status | `CollisionRecord` | Separated distance |
| `Collision.distanceBetweenRectangleCircle(...)` | Rect-circle distance & status | `CollisionRecord` | Separated distance |
| `Collision.lineIntersectsCircle(a, b, pos, rad)` | Line-circle ray-cast | `IntersectionResponse` | Hitscan hit detection |

### Easing & Animation (Numeric namespace)

Built on `Numeric.lerp()`. Easing functions apply non-linear interpolation:

```typescript
// Linear interpolation
t = 0.5;
value = Numeric.lerp(0, 100, t);  // 50

// Easing function (out-of-scope in this extract but documented in source)
// Cubic easing steepens at start/end for naturalistic motion
```

## Physics Constants

Physics behavior controlled via configuration objects. Bullets define ballistic properties:

### BaseBullet Definition Type (from common/src/utils/baseBullet.ts)

```typescript
type BaseBulletDefinition = {
    readonly damage: number              // Base damage per hit
    readonly obstacleMultiplier: number  // Damage vs obstacles (often < 1.0)
    readonly speed: number               // Pixels per tick
    readonly range: number               // Max distance before disappear
    readonly rangeVariance?: number      // Range randomization
    readonly shrapnel?: boolean           // Spawn child projectiles on impact
    readonly allowRangeOverride?: boolean // Allow items to override range
    readonly lastShotFX?: boolean         // Special effect on last round
    readonly noCollision?: boolean        // Ignore hitboxes
    readonly noReflect?: boolean          // Don't bounce off walls
    readonly ignoreCoolerGraphics?: boolean

    // Status effects
    readonly infection?: number           // Infection damae
    readonly teammateHeal?: number        // Heal value
    readonly enemySpeedMultiplier?: {
        duration: number
        multiplier: number
    }
    readonly removePerk?: ReferenceTo<PerkDefinition>

    // Visual tracer (bullet trail)
    readonly tracer?: {
        opacity?: number                  // Default 1
        width?: number                    // Default 1
        length?: number                   // Default 1
        color?: number                    // -1 for random
        saturatedColor?: number
        image?: string
        particle?: boolean
        spinSpeed?: number
        zIndex?: ZIndexes
    }

    // Visual trail (particle effects)
    readonly trail?: {
        interval: number                  // Spawn particles every N ticks
        amount?: number                   // Particles per spawn
        frame: string                     // Sprite frame
        scale: { min: number, max: number }
        alpha: { min: number, max: number }
        spreadSpeed: { /* ... */ }
    }
}
```

## Dependencies

### Depends on:
- **Nothing** — this is a foundational layer (no imports from other subsystems)

### Depended on by:
- [Game Objects (Server)](../game-objects-server/) — hitbox collision, layer checks
- [Game Objects (Client)](../game-objects-client/) — vector transforms, hitbox visualization
- [Game Loop](../game-loop/) — deterministic simulation via SeededRandom
- [Map](../map/) — procedural generation via SeededRandom, geometry for POI placement
- [Gas System](../gas/) — circle collision for gas radius expansion
- [Inventory](../inventory/) — item hitbox bounds, visual positioning
- [Networking](../networking/) — vector serialization in packets
- [Spatial Grid](../spatial-grid/) — vector distance calculations, AABB queries

## Known Issues & Gotchas

1. **Vector is readonly** — All operations return new Vector objects. Attempting to mutate `vec.x = 5` will fail or be ignored. This is by design for functional correctness.

2. **Polygon hitbox incomplete** — `PolygonHitbox` exists but some intersection methods throw "unsupported operation" errors. Safe fallback: convert to bounding rectangle with `toRectangle()`.

3. **SeededRandom is not cryptographic** — Uses simple LCG (linear congruential generator). Predictable to players who understand the algorithm. **Never** use for security (auth, RNG-based anti-cheat).

4. **Layer changes are discrete** — No smooth transition between layers. Objects pop when crossing stair layers; causes visual discontinuity if not handled by client renderer.

5. **Angle normalization wraps at ±π** — Angles outside ±π radians (±180°) are normalized. Can cause unexpected values: `normalize(π + 0.1)` → `-π + 0.1` (wraps around).

6. **Hitbox transform doesn't deep-clone** — `transform()` creates new shape but reuses Vector references if not explicitly cloned. Mutating the result's vectors affects inputs. Use `clone(true)` for deep copy.

7. **CollisionRecord distance is signed** — Negative distance = overlapping; positive distance = gap. Code often checks `if (record.distance < 0)` for collision.

8. **Group hitbox iteration unsorted** — `GroupHitbox.getIntersection()` returns first collision found, not deepest penetration (could fail in tight spaces).

9. **Layer predicates ignore object definitions** — `adjacentOrEqualLayer()` only checks numeric layer values. Use `equivLayer()` for definition-aware checks (respects `collideWithLayers` & `isStair` flags).

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Tech stack, system overview
- [Data Model](../../datamodel.md) — Vector type definition, Layer type definition

### Tier 2
- [Networking](../networking/) — How vectors are serialized in packets
- [Game Objects (Server)](../game-objects-server/) — Hitbox collision, layer checks during gameplay
- [Game Objects (Client)](../game-objects-client/) — Vector rendering, hitbox visualization
- [Game Loop](../game-loop/) — SeededRandom for deterministic simulation
- [Map Generation](../map/) — Uses SeededRandom for procedural POI/obstacle placement
- [Spatial Grid](../spatial-grid/) — Uses Vector distance, AABB queries
- [Gas System](../gas/) — Circle collision for expanding gas radius

### Tier 3
- (Under development) Game Loop — Tick Module
- (Under development) Map — Procedural Generation Module
- (Under development) Spatial Grid — Query Module
