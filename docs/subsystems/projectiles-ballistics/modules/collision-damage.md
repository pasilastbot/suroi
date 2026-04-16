# Projectiles & Ballistics — Collision & Damage Calculation

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/projectiles-ballistics/README.md -->
<!-- @source: server/src/objects/bullet.ts, server/src/objects/projectile.ts, server/src/objects/explosion.ts -->

## Purpose

Detailed mechanics of bullet/projectile collision detection, bounce physics, and damage application to players, obstacles, and buildings.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/bullet.ts` | Bullet trajectory, raycasting, reflection | High |
| `server/src/objects/projectile.ts` | Throwable physics, bounce, detonation damage | High |
| `server/src/objects/explosion.ts` | Explosion raycasting and multi-target damage calculation | High |
| `common/src/utils/baseBullet.ts` | Shared bullet collision detection logic | High |
| `common/src/utils/math.ts` | Geometry and line intersection algorithms | Medium |
| `server/src/objects/gameObject.ts` | Abstract damage() interface | Low |
| `server/src/objects/player.ts` | Damage application with armor reduction | High |

## Bullet Collision Detection

### Ray-Casting Algorithm

Each tick, bullets perform **line-based collision detection** (not point-continuous) to avoid tunneling through thin obstacles.

```typescript
// From server/src/objects/bullet.ts — update() method
update(): DamageRecord[] {
    const lineRect = RectangleHitbox.fromLine(
        this.position,
        Vec.add(this.position, Vec.scale(this.velocity, this.game.dt))
    );
    const { grid, dt } = this.game;
    
    // Broad-phase: grid query for objects in the bullet's path
    const objects = grid.intersectsHitbox(lineRect, this.layer);
    
    const records: DamageRecord[] = [];
    const damageMod = (this.modifiers?.damage ?? 1) / (this.reflectionCount + 1);
    
    for (const collision of this.updateAndGetCollisions(dt, objects)) {
        // Process each collision (sorted by distance)
        records.push({
            object,
            damage: damageMod * damageAmount * (obstacleMultiplier),
            weapon: this.sourceGun,
            source: this.shooter,
            position: this.position
        });
    }
    
    return records;
}
```

### Collision Detection (`updateAndGetCollisions`)

From `common/src/utils/baseBullet.ts`:

```typescript
updateAndGetCollisions(delta: number, objects: Iterable<GameObject>): BulletCollision[] {
    const oldPosition = this._oldPosition = Vec.clone(this.position);
    
    // Step 1: Update position
    this.position = Vec.add(this.position, Vec.scale(this.velocity, delta));
    
    // Step 2: Check max range (kill out-of-range bullets)
    if (Geometry.distanceSquared(this.initialPosition, this.position) > this.maxDistanceSquared) {
        this.dead = true;
        this.position = Vec.add(this.initialPosition, Vec.scale(this.direction, this.maxDistance));
    }
    
    if (this.definition.noCollision) return [];
    
    const collisions: BulletCollision[] = [];
    
    // Step 3: Iterate candidate objects from broad-phase query
    for (const object of objects) {
        // Skip non-damageable, dead, or already-hit objects
        if (!object.damageable || object.dead || this.collidedIDs.has(object.id)) continue;
        
        // Skip shooter (unless bullet can hit shooter: shrapnel or reflected)
        if (object.id === this.sourceID && !this.canHitShooter) continue;
        
        // Step 4: Line-based intersection
        const intersection = object.hitbox?.intersectsLine(oldPosition, this.position);
        
        if (intersection) {
            collisions.push({
                intersection,      // { point, normal }
                object,
                dealDamage: true,
                reflected: (object.isObstacle || object.isBuilding) 
                    && object.definition.reflectBullets
            });
        }
    }
    
    // Step 5: Sort by closest to initial position (not current)
    collisions.sort(
        (a, b) =>
            Geometry.distanceSquared(a.intersection.point, this.initialPosition)
            - Geometry.distanceSquared(b.intersection.point, this.initialPosition)
    );
    
    return collisions;
}
```

### Line-Hitbox Intersection

The core collision test uses `hitbox.intersectsLine(oldPosn, newPosn)` from [collision-hitbox](../../collision-hitbox/) subsystem. This returns:
- `null` if no intersection
- `{ point: Vector, normal: Vector }` if intersection found

**Line intersection algorithms:**
- **Circle hitbox:** distance from circle center to line ≤ radius
- **Rectangle hitbox:** AABB overlap with line's bounding box + edge tests
- **Polygon hitbox:** ray-edge intersection for each edge
- **Group hitbox:** recurse for each child, return closest hit

### Penetration & Reflection

After a collision hits:

```typescript
const reflected = (
    collision.reflected
    && this.reflectionCount < 3
    && !definition.noReflect
    && (definition.onHitExplosion === undefined || !definition.explodeOnImpact)
);

if (reflected) {
    // Bullet bounces off reflective surface (e.g., shield)
    // Spawn new bullet with reflected direction
    this.reflect(rotation ?? 0);
    this.reflected = true;
} 

this.dead = true;
break;  // Stop processing further collisions this tick
```

**Reflection Logic:**
1. Max 3 reflections per bullet (`reflectionCount < 3`)
2. Damage is **divided by (reflectionCount + 1)** — so 2x reflection ≈ 50% damage
3. Non-reflecting bullets die on first hit and stop processing collisions

## Bullet Damage Calculation

### Damage Formula (Player Impact)

```typescript
// From server/src/objects/bullet.ts
const damageAmount = definition.damage;  // Base damage from bullet definition

const finalDamage = (
    damageMod                           // (1.0 / (reflectionCount + 1))
    * damageAmount
    * (isObstacle ? definition.obstacleMultiplier : 1)  // For obstacles only
);

// Add to damage queue for processing at end of tick
records.push({
    object,
    damage: finalDamage,
    weapon: this.sourceGun,
    source: this.shooter,
    position: this.position
});
```

### Player Damage Application

From `server/src/objects/player.ts#2626`:

```typescript
override damage(params: DamageParams): void {
    if (this.invulnerable) return;
    
    let { amount } = params;
    
    // Armor reduction (helmet + vest)
    if (this.shield <= 0) {
        amount *= (1 - (
            (this.inventory.helmet?.damageReduction ?? 0) 
            + (this.inventory.vest?.damageReduction ?? 0)
        )) 
        * this.mapPerkOrDefault(PerkIds.LastStand, ({ damageReceivedMod }) => damageReceivedMod, 1);
    }
    
    // Apply the reduced damage
    this.piercingDamage({ amount, source, weaponUsed });
}
```

**Armor Formula:**
```
actualDamage = baseDamage × (1 - (helmetReduction + vestReduction)) × perkMultiplier
```

Example:
- Bullet: 50 damage
- Helmet: 0.25 reduction, Vest: 0.25 reduction
- Calculation: 50 × (1 - 0.5) × 1.0 = **25 damage**

### Damage Queuing (Critical!)

Damage is **not applied immediately** on collision. Instead, damage is accumulated in a queue and applied at the end of the tick:

```typescript
// From server/src/game.ts

// Tick N: All bullets collect collisions
let records: DamageRecord[] = [];
for (const bullet of this.bullets) {
    records = records.concat(bullet.update());  // Collects but doesn't apply
    
    if (bullet.dead) {
        // Handle explosion on impact
        if (!bullet.reflected && bullet.definition.onHitExplosion) {
            this.addExplosion(...);
        }
        this.bullets.delete(bullet);
    }
}

// End of Tick N: Apply all collected damage at once
for (const { object, damage, source, weapon, position } of records) {
    object.damage({
        amount: damage,
        source,
        weaponUsed: weapon,
        position
    });
    
    // Handle on-hit projectiles (projectiles that spawn on bullet impact)
    if (weapon.definition.ballistics.onHitProjectile) {
        const proj = this.addProjectile({
            position,
            definition: onHitProjectile,
            ...
        });
    }
}
```

**Why queue damage?**
- Prevents double-hits in single tick (e.g., shotgun pellets hitting same crate all damage it once)
- Ensures all bullets see the same game state on collision
- Matches client-side expectation (client sees crate destroyed, not taking multiple hits)

## Projectile Bounce Physics

### Obstacle Collision

From `server/src/objects/projectile.ts#update()`:

```typescript
// Collisions resolved in sorted order (closest first)
for (const collision of collisions) {
    const { object } = collision;
    
    // Stairs pass through (special case)
    if (object.isObstacle && object.definition.isStair) {
        object.handleStairInteraction(this);
        continue;
    }
    
    // Check height: projectile above obstacle?
    const height = object.height;
    if (this._height >= height) {
        this._obstaclesBelow.add(object);
        continue;  // Projectile passes over obstacle
    }
    
    // Actual collision: resolve position and bounce
    if (collision.isLine) {
        this.position = collision.position;  // Snap to collision point
    }
    
    if (!this.inAir) continue;  // Projectile already on ground
    
    // Bounce physics
    this._reflect(collision.normal);
    
    break;  // Stop processing collisions
}

private _reflect(normal: Vector): void {
    const length = Vec.len(this._velocity);
    const direction = Vec.scale(this._velocity, 1 / length);
    const dot = Vec.dotProduct(direction, normal);
    const newDir = Vec.add(Vec.scale(normal, dot * -2), direction);
    this._velocity = Vec.scale(newDir, length * 0.3);  // 30% elasticity
}
```

**Bounce Formula:**
```
v_reflected = v - 2(v · n) × n
v_final = v_reflected × 0.3  (30% energy retention)
```

### Height System (Z-Axis Physics)

Projectiles have a height (Z-axis) independent of position (X-Y):

```typescript
// Gravity and velocity
this._velocityZ -= GameConstants.projectiles.gravity * dt;  // gravity = 10
this._height += this._velocityZ * dt;
this._height = Numeric.clamp(this._height, 0, GameConstants.projectiles.maxHeight);

// Drag (friction) changes based on surface
const drag = this.definition.physics.drag ?? GameConstants.projectiles.drag;

if (onWater) speedDrag = drag.water;
else if (onIce) speedDrag = drag.ice;
else if (onFloor || sittingOnObstacle) speedDrag = drag.ground;
else speedDrag = drag.air;

// Apply drag
this._velocity = Vec.scale(this._velocity, 1 / (1 + dt * speedDrag));
```

**Drag Example:**
- Air drag: 0.7
- Ground drag: 3
- Water drag: 5
- Ice drag: varies

### Impact Damage

Projectile can deal damage on collision:

```typescript
if ((object.isObstacle || object.isPlayer) && this.definition.impactDamage) {
    const damage = this.definition.impactDamage
        * (object.isPlayer ? 1 : this.definition.obstacleMultiplier ?? 1);
    
    object.damage({ 
        amount: damage, 
        source: this.owner, 
        weaponUsed: this.source 
    });
}
```

## Explosion Damage Calculation

### Raycasting Algorithm

Explosions use **radial raycasting** to find damaged targets, preventing damage through walls:

```typescript
// From server/src/objects/explosion.ts#explode()
explode(): void {
    const definition = this.definition;
    const objects = this.game.grid.intersectsHitbox(
        new CircleHitbox(definition.radius.max * 2, this.position), 
        this.layer
    );
    
    const damagedObjects = new Set<number>();
    
    // Calculate angular step size using uniform distance formula
    // GameConstants.explosionRayDistance = 2
    const step = Math.acos(1 - ((GameConstants.explosionRayDistance / definition.radius.max) ** 2) / 2);
    
    // Raycasts: -π to π at `step` intervals
    for (let angle = -Math.PI; angle < Math.PI; angle += step) {
        const lineCollisions: Array<{
            readonly object: GameObject
            readonly pos: Vector
            readonly squareDistance: number
        }> = [];
        
        const lineEnd = Vec.add(
            this.position, 
            Vec.fromPolar(angle, definition.radius.max)
        );
        
        // Step 1: Test each object against this ray
        for (const object of objects) {
            if (object.dead || !object.hitbox) continue;
            
            const intersection = object.hitbox.intersectsLine(
                this.position, 
                lineEnd
            );
            if (intersection) {
                lineCollisions.push({
                    pos: intersection.point,
                    object,
                    squareDistance: Geometry.distanceSquared(
                        this.position, 
                        intersection.point
                    )
                });
            }
        }
        
        // Step 2: Sort by distance (closest blocks further objects)
        lineCollisions.sort((a, b) => a.squareDistance - b.squareDistance);
        
        // Step 3: Apply damage, stopping at solid obstacles
        const { min, max } = definition.radius;
        for (const collision of lineCollisions) {
            const object = collision.object;
            const { isPlayer, isObstacle, isBuilding, isLoot, isProjectile } = object;
            
            if (!damagedObjects.has(object.id)) {
                damagedObjects.add(object.id);
                const dist = Math.sqrt(collision.squareDistance);
                
                // **Damage Formula**
                const damage = this.damageMod * definition.damage
                    * (isObstacle ? definition.obstacleMultiplier : 1)
                    * (isPlayer ? perkMultiplier : 1)
                    * ((dist > min) ? (max - dist) / (max - min) : 1);
                
                object.damage({
                    amount: damage,
                    source: this.source,
                    weaponUsed: this
                });
            }
            
            // Stop ray at solid obstacles (not stairs)
            if ((isObstacle && !object.definition.noCollisions && !object.definition.isStair)
                || (isBuilding && !object.definition.noCollisions)) {
                break;
            }
        }
    }
}
```

### Explosion Damage Falloff Formula

```
damage = baseDamage × damageMod × obstacleMultiplier × perkMultiplier × falloff

where:
  falloff = (max_radius - distance) / (max_radius - min_radius)  [if distance > min_radius]
          = 1.0                                                   [if distance ≤ min_radius]

clamp falloff to [0, 1]
```

**Example:**
- Base explosion damage: 100
- Min radius: 5, Max radius: 30
- Target at distance 17.5

```
falloff = (30 - 17.5) / (30 - 5) = 12.5 / 25 = 0.5
damage = 100 × 1.0 × 0.5 = 50
```

### Ray Angular Spacing

The ray step size ensures **uniform point distribution across circle**:

```typescript
const step = Math.acos(1 - ((explosionRayDistance / definition.radius.max) ** 2) / 2);
```

Where `explosionRayDistance = 2` (from `GameConstants`).

This is derived from the **chord-length formula**. For uniform ~2-unit spacing along the circumference:

```
arccos(1 - c²/2r²) = step
where c = chord length (2 units), r = radius
```

For a 30-unit radius explosion:
```
step = acos(1 - 4/1800) ≈ 0.0667 radians ≈ 3.8°
rays ≈ 2π / 0.0667 ≈ 94 rays total
```

For line-based damage:
- Each ray goes from explosion center to `max_radius`
- Hits are sorted by distance, closest first
- Solid obstacles block further targets on that ray

### Knockback on Explosion

```typescript
// From explosion.ts
if (isLoot || isProjectile) {
    const multiplier = isProjectile ? 0.002 : 0.01;
    object.push(
        Angle.betweenPoints(object.position, this.position),
        (max - dist) * multiplier
    );
}
```

**Knockback on loot:** `(maxRadius - distance) × 0.01`
**Knockback on projectile:** `(maxRadius - distance) × 0.002`

Direction: from explosion center toward target.

## Damage Event Lineage

### Sequence of Events (Single Tick)

```
Tick N begins (0 ms)
├─ Update phase (0-15 ms):
│  ├─ Bullet.update() → updateAndGetCollisions() → DamageRecord[]
│  ├─ Projectile.update() → collision → impactDamage queued
│  └─ Collision data collected, NOT applied yet
│
├─ Explosion trigger phase (15-20 ms):
│  ├─ Bullet hits wall → onHitExplosion → Explosion.explode()
│  ├─ Raycast for targets
│  └─ Damage queued
│
├─ Damage application phase (20-25 ms):
│  ├─ For each DamageRecord in queue:
│  │  └─ object.damage(amount, source, weapon)
│  └─ Player health/shield reduced
│
└─ Network phase (25 ms):
   ├─ Build UpdatePacket with dirty objects
   └─ Send to clients (40 TPS = 25 ms per tick)
```

### Same-Tick Hit Prevention

Without the queue system, shotgun pellets would inconsistently hit:
- **Server:** Pellet 1 hits crate, destroys it; Pellet 2-7 pass through (no object)
- **Client:** All 7 pellets hit crate at same time (no destruction yet)
- Result: Desync (client says crate alive, server says dead)

**Solution:** Queue all damage, apply at tick end. Now:
- All bullets see crate alive during collision checks
- All damage is applied to crate in sequence at tick end
- Client matches server (receives destruction packet after seeing hit damage)

## Known Gotchas & Caveats

### 1. **Ray Doesn't Account for Bullet Size**

Bullets are treated as **infinitesimal rays**, not cylinders. A thin bullet can tunnel through narrow gaps that a larger projectile couldn't fit through.

```typescript
// From baseBullet.ts — no radius/thickness used in intersectsLine()
const intersection = object.hitbox?.intersectsLine(oldPosition, this.position);
```

**Gotcha:** A 0.25-unit-wide gap will block a thrown grenade but not a bullet ray.

### 2. **Reflection Damage Loss is Multiplicative, Not Per-Bounce**

Damage is divided once at the start based on total reflection count:

```typescript
const damageMod = (this.modifiers?.damage ?? 1) / (this.reflectionCount + 1);
```

- 1st shot: `damageMod = damage / 1 = full damage`
- 1st reflection: `damageMod = damage / 2 = 50%`
- 2nd reflection: `damageMod = damage / 3 = 33%`

**Gotcha:** Reflecting 3 times = 25% damage (1/4), not 50% × 50% × 50% (12.5%).

### 3. **Penetration Doesn't Work (Most Bullets)**

Despite the code checking `definition.penetration`, most bullets have `penetration = 0` or `undefined`.

**Current behavior:**
- If penetration == 0 → bullet dies on hit, no further collisions
- If penetration > 0 → bullet reflects instead (uses reflection system)

**Gotcha:** "Penetration" is actually "reflection". A bullet can't pass *through* an obstacle; it bounces off.

### 4. **Explosion Ray Blocking is "First Solid Hit"**

Obstacles stop the ray immediately; there's no damage through multiple thin walls:

```typescript
if ((isObstacle && !object.definition.noCollisions && !object.definition.isStair)
    || (isBuilding && !object.definition.noCollisions)) {
    break;  // Stop this ray
}
```

**Gotcha:** Hiding behind a wall protects you completely from explosion damage on that ray. No "splash damage over walls."

### 5. **Same-Tick Multi-Hit Possible**

If two bullets fire in the same tick (e.g., shotgun), both can hit the same target:

```typescript
// Both bullets check collision independently
for (const bullet of this.bullets) {
    records = records.concat(bullet.update());
}

// Then all damage is applied
for (const { object, damage } of records) {
    object.damage({ amount: damage, ... });
}
```

**Gotcha:** Single player can take damage from multiple sources in one tick. No "hit cooldown" per target per tick.

### 6. **Armor Reduction is Additive, Not Multiplicative**

```typescript
amount *= (1 - (helmet.damageReduction + vest.damageReduction))
```

- Helmet (0.25) + Vest (0.25) = **50% reduction** (not 43.75%)
- No "stacking diminishment"

**Gotcha:** Two armor pieces are more effective together than applied sequentially.

### 7. **HeadShot Multiplier in Line of Sight Subsystem**

Headshot detection is **not** in bullet/collision code; it's handled by the [visibility-los](../../visibility-los/) subsystem or melee combat system elsewhere.

**Gotcha:** Bullet collision doesn't know about headshots—damage is uniform across hitbox.

### 8. **Projectile Height is Separate from Position**

Height (Z-axis) doesn't prevent collision in X-Y plane; it only affects obstacle pass-through:

```typescript
if (this._height >= height) {
    this._obstaclesBelow.add(object);
    continue;  // Pass over, don't collide
}
```

**Gotcha:** A projectile can "float" above ground (height > 0) but still collide with ground-level players. Height is checked only for *obstacle* collisions.

### 9. **Damage Modifiers Don't Serialize to Client**

The damage modifier is explicitly not sent to client:

```typescript
// From baseBullet.ts#serialize()
// don't care about damage
// don't care about dtc
```

**Gotcha:** Client-side bullet visualization sees different damage than server applies. Perk multipliers, DTC (Damage to Crate), etc. are server-only secrets.

## Performance Considerations

### Spatial Grid Culling

Bullets use broad-phase spatial grid queries:

```typescript
const objects = grid.intersectsHitbox(lineRect, this.layer);
```

Grid cell size: 32 units (from `GameConstants.gridSize`). Bullets only check objects in cells their path intersects.

**Cost:** O(bullets × cells × objects_per_cell)

### Explosion Raycasting Cost

Explosion damage is O(rays × objects_per_radius):

```typescript
// ~94 rays per 30-unit explosion × ~100 objects = 9,400 checks
for (let angle = -Math.PI; angle < Math.PI; angle += step) {
    for (const object of objects) {
        const intersection = object.hitbox.intersectsLine(center, lineEnd);
    }
}
```

**Mitigation:** Large explosions use very small angular step (2-unit chord length), causing many rays. Multiple explosions (e.g., grenade cluster) can spike CPU.

### Bullet Pooling

Bullets are allocated and freed frequently. The system doesn't use object pools yet—each bullet is `new Bullet()`.

**Gotcha:** High fire rate ≈ high GC pressure on server.

## Related Documents

### Tier 2
- [Projectiles & Ballistics](../README.md) — Full system overview, throwable definitions, weapon configurations
- [Collision & Hitbox](../../collision-hitbox/) — Hitbox shapes, line-to-shape intersection tests
- [Spatial Grid](../../spatial-grid/) — Broad-phase culling, grid cell queries
- [Game Loop](../../game-loop/) — Tick scheduling, update phase timing

### Tier 3
- [game-loop/modules/update-phase.md](../../game-loop/modules/update-phase.md) — Where bullet.update() is called in the tick sequence
- [server-utilities/modules/grid.md](../../server-utilities/modules/grid.md) — Grid.intersectsHitbox() implementation
- [visibility-los/modules/raycast.md](../../visibility-los/modules/raycast.md) — Client-side headshot visualization (mirrors server collision)

### Dependencies
- `@common/definitions/bullets` — Bullet property definitions (damage, range, obstacleMultiplier)
- `@common/definitions/explosions` — Explosion definitions (radius, damage, obstacleMultiplier)
- `common/src/utils/hitbox.ts` — Circle, Rectangle, Polygon, Group hitbox intersections
- `common/src/constants.ts` — GameConstants.explosionRayDistance, explosionMaxDistSquared, gridSize
