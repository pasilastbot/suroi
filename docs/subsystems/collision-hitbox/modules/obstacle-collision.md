# Obstacle Collision & Response

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/objects/obstacle.ts, server/src/objects/player.ts, common/src/utils/hitbox.ts -->

## Purpose

This module handles collision detection and response between players/bullets and obstacles. It covers obstacle types with varying collision properties, hitbox shapes, destructible behavior, layer-aware collision, and the push-out algorithm that prevents interpenetration.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/obstacle.ts` | Obstacle class definition, health system, collision properties | High |
| `server/src/objects/player.ts` | Player collision detection loop, collision resolution steps | High |
| `common/src/utils/hitbox.ts` | Hitbox types (Circle, Rect, Group), collision testing, push-out algorithm | High |
| `common/src/utils/math.ts` | SAT collision detection, intersection response calculations | Very High |
| `common/src/definitions/obstacles.ts` | ObstacleDefinition schema with collision flags | Medium |

---

## Obstacle Types

Obstacles in Suroi have varying collision properties based on their definition. The collision behavior is determined by:

### Definition Properties

| Property | Type | Effect |
|----------|------|--------|
| `noCollisions` | boolean | If true, obstacle has no collision (e.g., decorations, foliage overlays) |
| `noCollisionAfterDestroyed` | boolean | Becomes non-collidable when destroyed (excludes windows) |
| `noHitEffect` | boolean | Damage doesn't trigger visual impact effect, but *still collidable* |
| `health` | number | Durability; when health ≤ 0, obstacle is destroyed |
| `indestructible` | boolean | Cannot be damaged by any weapon |
| `impenetrable` | boolean | Cannot be damaged unless weapon has piercing multiplier or melee hardness exceeds obstacle hardness |
| `hitbox` | Hitbox | Collision bounds (CircleHitbox, RectangleHitbox, or GroupHitbox) |
| `spawnHitbox` | Hitbox (optional) | Separate hitbox for spawn placement; when omitted, uses `hitbox` |
| `isDoor` | boolean | Door mechanics with open/closed hitbox swapping |
| `isStair` | boolean | Layer transitions when player overlaps (no position push-out) |
| `isWindow` | boolean | Remains collidable when destroyed (unlike other obstacles) |
| `material` | string | Material type (wood, stone, metal, pumpkin, tree, etc.) for damage resistance and audio |
| `allowFlyover` | FlyoverPref | `Always` (height 0.2), `Sometimes` (height 0.5), or Infinity (blocks flyover) |
| `hardness` | number | Melee weapon hardness threshold to destroy |

### Obstacle Collidable State

From [server/src/objects/obstacle.ts](server/src/objects/obstacle.ts#L25):

```typescript
export class Obstacle extends BaseGameObject.derive(ObjectCategory.Obstacle) {
    override readonly damageable = true;
    health: number;
    readonly maxHealth: number;
    collidable: boolean;  // Can be toggled during destruction
    override hitbox: Hitbox;
    
    constructor(...) {
        this.collidable = !definition.noCollisions;
        if (dead) {
            if (!this.definition.isWindow || this.definition.noCollisionAfterDestroyed) {
                this.collidable = false;  // Becomes non-collidable
            }
        }
    }
}
```

---

## Collision Testing

### Broad-Phase: Spatial Grid Culling

Before narrow-phase detection, the game uses a spatial grid to cull distant objects:

From [server/src/objects/player.ts](server/src/objects/player.ts#L1282):

```typescript
// Find and resolve collisions
this.nearObjects = this.game.grid.intersectsHitbox(this._hitbox, this.layer);
```

The grid returns only obstacles on the player's layer within the player's hitbox bounds, reducing O(n) to roughly O(log n).

### Narrow-Phase: Hitbox Intersection Testing

Suroi supports three hitbox types: `CircleHitbox`, `RectangleHitbox`, and `GroupHitbox`.

#### Circle-Circle Intersection

From [common/src/utils/math.ts](common/src/utils/math.ts#L629):

```typescript
circleCircleIntersection(centerA: Vector, radiusA: number, centerB: Vector, radiusB: number): CollisionResponse {
    const r = radiusA + radiusB;
    const toP1 = Vec.sub(centerB, centerA);
    const distSqr = Vec.squaredLen(toP1);

    return distSqr < r * r
        ? {
            dir: Vec.normalizeSafe(toP1),  // Push direction
            pen: r - Math.sqrt(distSqr)    // Penetration depth
        }
        : null;
}
```

**Algorithm:**
1. Calculate sum of radii (separation distance if touching)
2. Calculate squared distance between centers
3. If centers are closer than sum of radii → collision
4. Return direction (normalized vector from A to B) and penetration depth

#### Rectangle-Circle Intersection (SAT-based)

From [common/src/utils/math.ts](common/src/utils/math.ts#L641):

```typescript
rectCircleIntersection(min: Vector, max: Vector, pos: Vector, radius: number): CollisionResponse {
    // Case 1: Circle center inside rectangle
    if (min.x <= pos.x && pos.x <= max.x && min.y <= pos.y && pos.y <= max.y) {
        const halfDimension = Vec.scale(Vec.sub(max, min), 0.5);
        const p = Vec.sub(pos, Vec.add(min, halfDimension));
        const xp = Math.abs(p.x) - halfDimension.x - radius;
        const yp = Math.abs(p.y) - halfDimension.y - radius;

        return xp > yp  // Push out on X or Y axis
            ? {
                dir: Vec(p.x > 0 ? -1 : 1, 0),
                pen: -xp
            }
            : {
                dir: Vec(0, p.y > 0 ? -1 : 1),
                pen: -yp
            };
    }

    // Case 2: Circle center outside rectangle
    const dir = Vec.sub(
        Vec(
            Numeric.clamp(pos.x, min.x, max.x),
            Numeric.clamp(pos.y, min.y, max.y)
        ),
        pos
    );
    const dstSqr = Vec.squaredLen(dir);

    if (dstSqr < radius * radius) {
        const dst = Math.sqrt(dstSqr);
        return {
            dir: Vec.normalizeSafe(dir),
            pen: radius - dst
        };
    }

    return null;
}
```

**Algorithm (Separating Axis Theorem for circle vs rectangle):**
1. **If circle center inside rectangle:** Use distance from center to nearest edge
   - Project circle center onto each axis (X, Y)
   - Find which axis requires less push-out
   - Return direction along that axis
2. **If circle center outside rectangle:** Find closest point on rectangle using clamping
   - Clamp circle center to rectangle bounds
   - Distance from clamped point to circle center is overlap depth
   - Direction is normalized vector from clamped point to circle center

#### Rectangle-Rectangle Intersection (SAT-based)

From [common/src/utils/math.ts](common/src/utils/math.ts#L769):

```typescript
rectRectIntersection(min0: Vector, max0: Vector, min1: Vector, max1: Vector): CollisionResponse {
    const e0 = Vec.scale(Vec.sub(max0, min0), 0.5);  // Half-extents of rect 0
    const e1 = Vec.scale(Vec.sub(max1, min1), 0.5);  // Half-extents of rect 1
    const n = Vec.sub(
        Vec.add(min1, e1),      // Center of rect 1
        Vec.add(min0, e0)       // Center of rect 0
    );
    const xo = e0.x + e1.x - Math.abs(n.x);  // X-axis overlap
    const yo = e0.y + e1.y - Math.abs(n.y);  // Y-axis overlap

    return xo > 0 && yo > 0  // Overlapping on both axes
        ? xo > yo
            ? {
                dir: Vec(Math.sign(n.x) || 1, 0),
                pen: xo
            }
            : {
                dir: Vec(0, Math.sign(n.y) || 1),
                pen: yo
            }
        : null;
}
```

**Algorithm (SAT):**
1. Calculate half-extents for both rectangles
2. Calculate distance between centers
3. Calculate overlap on X and Y axes:
   - X overlap = (half-width-0 + half-width-1) - |distance.x|
   - Y overlap = (half-height-0 + half-height-1) - |distance.y|
4. If overlap > 0 on both axes → collision
5. Push along axis with minimum overlap
6. Direction sign indicates which direction to push (determined by center-to-center distance)

### Group Hitbox Collision

Group hitboxes contain multiple child hitboxes and test collision against each:

From [common/src/utils/hitbox.ts](common/src/utils/hitbox.ts#L235):

```typescript
override collidesWith(that: Hitbox): boolean {
    return this.hitboxes.some(hitbox => hitbox.collidesWith(that));
}
```

---

## Collision Response

### Push-Out Algorithm

When a collision is detected, the moving object (player) is pushed out of the static obstacle. The collision response is stored as:

```typescript
export type CollisionResponse = {
    readonly dir: Vector      // Normalized direction to push
    readonly pen: number      // Penetration depth (how much overlap)
} | null;
```

### CircleHitbox Resolution

From [common/src/utils/hitbox.ts](common/src/utils/hitbox.ts#L283):

```typescript
override resolveCollision(that: Hitbox): void {
    switch (that.type) {
        case HitboxType.Circle: {
            const collision = Collision.circleCircleIntersection(
                this.position, this.radius, that.position, that.radius
            );
            if (collision) {
                // Push this circle away from that circle
                this.position = Vec.sub(this.position, Vec.scale(collision.dir, collision.pen));
            }
            break;
        }
        case HitboxType.Rect: {
            const collision = Collision.rectCircleIntersection(
                that.min, that.max, this.position, this.radius
            );
            if (collision) {
                // Push this circle away from the rectangle
                this.position = Vec.sub(this.position, Vec.scale(collision.dir, collision.pen));
            }
            break;
        }
    }
}
```

**Algorithm:**
1. Calculate collision response (direction + penetration depth)
2. Apply push: `position -= collision.dir * collision.pen`
3. This moves the circle away from the obstacle until no longer overlapping

### RectangleHitbox Resolution

From [common/src/utils/hitbox.ts](common/src/utils/hitbox.ts#L403):

```typescript
override resolveCollision(that: Hitbox): void {
    switch (that.type) {
        case HitboxType.Rect: {
            const collision = Collision.rectRectIntersection(
                this.min, this.max, that.min, that.max
            );
            if (collision) {
                // Transform (translate) the entire rectangle
                const rect = this.transform(Vec.scale(collision.dir, -collision.pen));
                this.min = rect.min;
                this.max = rect.max;
            }
            break;
        }
    }
}
```

**Algorithm:**
1. Calculate collision response
2. Transform the rectangle by pushing it away: `transform(direction * -penetration)`
3. Update min/max bounds to reflect new position

### Multi-Step Collision Resolution

Players resolve collisions over multiple steps (0–10 iterations) to handle complex scenarios:

From [server/src/objects/player.ts](server/src/objects/player.ts#L1283):

```typescript
for (let step = 0; step < 10; step++) {
    let collided = false;

    for (const potential of this.nearObjects) {
        if (potential.collidable && potential.hitbox?.collidesWith(this._hitbox)) {
            if (isObstacle && potential.definition.isStair) {
                // Handle stairs (layer transition)
                potential.handleStairInteraction(this);
            } else if (!this._noClip) {
                collided = true;
                this._hitbox.resolveCollision(potential.hitbox);  // Push out
            }
        }
    }

    if (!collided) break;  // No more collisions, done
}
```

**Why 10 steps?**
- Each step resolves one collision at a time
- Multiple simultaneous collisions require multiple iterations
- Breaks early if no collision occurs on current step
- Prevents infinite loops in pathological cases

---

## Destructible Obstacles

### Health & Scaling

As an obstacle takes damage, its visual scale decreases from `maxScale` → `scale.destroy`:

From [server/src/objects/obstacle.ts](server/src/objects/obstacle.ts#L185):

```typescript
health: number;
readonly maxHealth: number;
readonly maxScale: number;

damage(params: DamageParams & { position?: Vector }): void {
    if (this.health === 0 || definition.indestructible) return;
    
    this.health -= amount;
    this.setPartialDirty();  // Network update

    const dead = this.health <= 0;
    if (!dead) {
        const oldScale = this.scale;
        
        // Interpolate scale from maxScale to destroyScale
        const destroyScale = definition.scale?.destroy ?? 1;
        this.scale = (this.health / this.maxHealth) * (this.maxScale - destroyScale) + destroyScale;
        
        // Scale hitbox proportionally
        this.hitbox.scale(this.scale / oldScale);
    }
}
```

**Scale Formula:**
```
newScale = (health / maxHealth) * (maxScale - destroyScale) + destroyScale
```

### Destruction Effects

When an obstacle dies, several systems trigger:

1. **Collidability:** Most obstacles become non-collidable (except windows)
   ```typescript
   if (!this.definition.isWindow || this.definition.noCollisionAfterDestroyed) {
       this.collidable = false;
   }
   ```

2. **Explosions:** Obstacles with `definition.explosion` create explosions
   ```typescript
   if (definition.explosion !== undefined && source instanceof BaseGameObject) {
       this.game.addExplosion(definition.explosion, this.position, source, this.layer);
   }
   ```

3. **Particles:** Visual destruction particles spawn
   ```typescript
   if (definition.particlesOnDestroy !== undefined) {
       this.game.addSyncedParticles(definition.particlesOnDestroy, this.position, this.layer);
   }
   ```

4. **Loot:** Items drop from destroyed obstacles
5. **Building Cascade:** Wall destruction triggers ceiling damage in parent building

---

## Layer-Based Collision

### Stair Interaction (Layer Transitions)

Stairs (`definition.isStair = true`) don't push players out; instead, they transition players between layers:

From [server/src/objects/obstacle.ts](server/src/objects/obstacle.ts#L413):

```typescript
/**
 * Resolves the interaction between a given game object or bullet and this stair.
 * Prerequisite: `this.definition.isStair` && `gameObject.hitbox.collidesWith(this.hitbox)`
 */
handleStairInteraction(object: GameObject | Bullet): void {
    object.layer = resolveStairInteraction(
        this.definition,
        this.rotation,
        this.hitbox as RectangleHitbox,
        this.layer,
        object.position
    );
}
```

The player's layer is updated based on their position relative to the stair's hitbox sides, allowing vertical movement without position push-out.

### Layer Filtering in Grid Lookup

The spatial grid retrieves objects on the same layer as the querying object:

```typescript
this.nearObjects = this.game.grid.intersectsHitbox(this._hitbox, this.layer);
```

This ensures obstacles on different layers (e.g., ground floor vs basement) don't interfere.

---

## Obstacle Groups

Some obstacles have multiple collision components for complex shapes:

### GroupHitbox

A `GroupHitbox` contains multiple child hitboxes (lists of `CircleHitbox` and `RectangleHitbox`):

From [common/src/utils/hitbox.ts](common/src/utils/hitbox.ts#L560):

```typescript
export class GroupHitbox extends BaseHitbox<HitboxType.Group> {
    hitboxes: ShapeHitbox[];

    override collidesWith(that: Hitbox): boolean {
        return this.hitboxes.some(hitbox => hitbox.collidesWith(that));
    }

    override resolveCollision(that: Hitbox): void {
        that.resolveCollision(this);  // Delegate to other hitbox
    }
}
```

### Door Hitbox Swapping

Doors have two hitboxes that swap based on state:

From [server/src/objects/obstacle.ts](server/src/objects/obstacle.ts#L60):

```typescript
door?: {
    isOpen: boolean;
    closedHitbox: Hitbox;
    openHitbox: Hitbox;
    openAltHitbox?: Hitbox;  // Swivel doors have two open positions
};
```

When a door opens, its collision hitbox changes from `closedHitbox` → `openHitbox`, allowing players through.

---

## Push Resolution

### Single Collision Step

A single collision step performs narrow-phase testing and resolution:

```
for each nearObject in spatial grid:
    if nearObject.collidable && collidesWithPlayer:
        if nearObject.isStair:
            handleStairInteraction(player)
        else:
            hitbox.resolveCollision(nearObject.hitbox)  // Push player out
```

### Multi-Collision Handling

When a player collides with multiple obstacles simultaneously, the algorithm resolves them iteratively:

```
Step 1: Player collides with obstacle A → push out
Step 2: After push-out, check again; if still colliding with B → push out
Step 3: Continue until no new collisions occur (max 10 steps)
```

This prevents players from getting stuck between two obstacles.

---

## Multiple Collisions

### Simultaneous Collision Loop

The 10-step loop in player collision handles cases where multiple obstacles are involved:

From [server/src/objects/player.ts](server/src/objects/player.ts#L1283):

```typescript
for (let step = 0; step < 10; step++) {
    let collided = false;

    for (const potential of this.nearObjects) {
        // ... collision checks ...
        if (someCollision) {
            collided = true;
            this._hitbox.resolveCollision(potential.hitbox);
        }
    }

    if (!collided) break;  // Exit early if no collisions this step
}
```

**Example Scenario:**
- Player walks into a corner between two walls
- Step 1: Player collides with wall A, pushes out in +X direction
- Step 2: Now player overlaps wall B, pushes out in +Y direction
- Step 3: No more collisions, loop breaks

**Limitation:** If after 10 steps collisions still exist, the loop exits. This prevents deadlocks but can cause players to clip into obstacles if they approach fast enough.

---

## Performance

### Broad-Phase: Spatial Grid Culling

The game uses a spatial hash grid (`Grid` class in `server/src/utils/grid.ts`) to partition the map:

**Grid Cell Size:** 32 units (from `GameConstants`)

**Query Complexity:** O(log n) for fetching nearby objects instead of O(n) full-world collision checks

```typescript
// Instead of checking all ~1000 obstacles on map:
const nearObjects = this.game.grid.intersectsHitbox(this._hitbox, this.layer);

// Only check ~5–20 objects in adjacent grid cells
for (const potential of nearObjects) { ... }
```

### Narrow-Phase: SAT Collision Tests

Separating Axis Theorem (SAT) tests are O(1) per pair:

- **Circle-Circle:** 3 distance calculations, 1 sqrt (if colliding)
- **Rect-Circle:** 2 clamps + distance calculation
- **Rect-Rect:** 6 arithmetic operations (half-extents + overlaps)

### Multi-Step Resolution Cost

The worst case is a player colliding with many obstacles:
- 10 steps × ~20 nearby objects = ~200 collision checks per player per tick
- With 100 players = 20,000 checks/tick ... but grid culling reduces this dramatically

### Tick Budget

At 40 TPS (25 ms per tick), collision must complete in <1 ms to not stall gameplay.

---

## Network Sync

### Obstacle State Serialization

Obstacle damage and destruction state is synchronized to clients via `partialDirtyObjects`:

| Field | Serialized | Frequency |
|-------|-----------|-----------|
| Position | Yes | Every tick (if moved) |
| Health | Yes (6 bytes partial) | When damaged |
| Scale | Yes | When health changes (via scale formula) |
| Collidable | Yes | When destroyed |
| Dead | Yes | When health ≤ 0 |

From [server/src/objects/obstacle.ts](server/src/objects/obstacle.ts#L25):

```typescript
override readonly fullAllocBytes = 10;     // Full state size
override readonly partialAllocBytes = 6;   // Damage state size
```

**Network Update Packet:**
- Full update (10 bytes): position, health, scale, rotation, layer
- Partial update (6 bytes): position + health (when damaged only)

### Client-Side Rendering

Clients apply the same scale formula and visual destruction effects to locally-predicted obstacles.

---

## Known Gotchas

### 1. Push-Out into Adjacent Obstacle

**Problem:** Resolving collision with obstacle A pushes player into obstacle B

**Example:**
```
[Obstacle A]
     ↓ (push force from A)
     [Player] → collides with [Obstacle B]
```

**Mitigation:** The 10-step loop re-checks collisions immediately after push-out. On the next iteration, collision with B is detected and corrected.

**Not Guaranteed:** If two obstacles are closer than the combined push distances, a player can still briefly clip.

### 2. Corner Clipping

**Problem:** Player moving diagonally into a sharp corner gets pushed partially into the wall

**Example:**
```
    [Obstacle A]
    ↓
[Player] → into corner
    ↓
    [Obstacle B]
```

**Why:** SAT pushes along the axis with minimum overlap. If pushing along X moves player into Y-axis bounds of another obstacle, clipping occurs.

**Mitigation:** Multi-step loop resolves this in subsequent iterations, but there's a visual flicker.

### 3. Ceiling Collision Edge Case

**Problem:** Players standing on ground-level obstacles can collide with obstacles above on different layers

**Why:** Layer filtering only applies to grid queries, not to subsequent collision tests on pushed objects.

**Status:** Rare in practice because stairs handle vertical transitions. Only occurs with unintended layer overlaps.

### 4. TypeScript Cast Issues in Destruction

From [server/src/objects/obstacle.ts](server/src/objects/obstacle.ts#L280):

```typescript
this.game.addExplosion(
    definition.explosion,
    this.position,
    source,
    this.layer,
    weaponIsItem ? weaponUsed : (weaponUsed as Explosion)?.weapon
    //                           ^^^^^^^^^^^^^^^^^^^^^^^^^
    // FIXME this cast to Explosion is not ideal
);
```

The weapon can be a gun item or an explosion. In the explosion case, we extract the original weapon. This cast is not type-safe and could fail if weaponUsed is neither.

### 5. Non-Rectangular Door Hitboxes Throw Error

From [server/src/objects/obstacle.ts](server/src/objects/obstacle.ts#L477):

```typescript
toggleDoor(player?: Player): void {
    if (!this.door) return;
    if (!(this.hitbox instanceof RectangleHitbox)) {
        throw new Error("Door with non-rectangular hitbox");
    }
}
```

Doors must have rectangular hitboxes. Circular or grouped door hitboxes will crash.

---

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Collision & Hitbox subsystem overview
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — System architecture
- **Tier 2 (Spatial Grid):** [../spatial-grid/README.md](../spatial-grid/README.md) — Grid internals & performance
- **Tier 2 (Game Loop):** [../../game-loop/README.md](../../game-loop/README.md) — Tick execution & collision timing
- **Tier 3 (Spatial Grid):** [../spatial-grid/modules/grid-query.md](../spatial-grid/modules/grid-query.md) — Grid cell queries
- **Patterns:** [../patterns.md](../patterns.md) — SAT collision patterns, hitbox composition patterns
