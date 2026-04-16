# Projectiles & Ballistics

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/objects/bullet.ts, server/src/objects/projectile.ts, server/src/objects/explosion.ts -->

## Purpose

Server-side ballistic physics and collision detection: gun bullets, thrown grenades, and explosions. Handles projectile trajectories, collision detection with game world objects, damage application, knockback, and synchronized network effects.

## Key Files & Entry Points

| Class | File | Responsibility |
|-------|------|-----------------|
| **Bullet** | `server/src/objects/bullet.ts` | Hitscan projectiles from firearms; straight-line trajectory, collision detection, impact effects |
| **Projectile** | `server/src/objects/projectile.ts` | Arc-based physics for grenades and throwables; gravity, bounce, fuse mechanics, detonation |
| **Explosion** | `server/src/objects/explosion.ts` | Radial damage and knockback on detonation; ray-casting to all nearby objects, falloff calculation |
| **BaseBullet** | `common/src/utils/baseBullet.ts` | Shared bullet configuration schema; tracer, trail, modifiers, range |

## Architecture

### Bullet Lifecycle

```
1. GunItem.use() fires weapon
   └─> game.addBullet(gunItem, player, { position, rotation })
   
2. Bullet created with:
   - position: gun muzzle position
   - rotation: aiming angle
   - velocity: speed * direction
   - definition: BulletDefinition from gun definition
   
3. Each game tick (25ms):
   - Create line hitbox from last position to new position
   - Grid query all objects in that line
   - Call updateAndGetCollisions() to check intersections
   
4. On collision:
   - Check collision.dealDamage flag
   - Build DamageRecord with (object, damage, weapon, source, position)
   - If onHitExplosion defined: add explosion to game
   - If reflected: create new reflected bullet with reflectionCount++
   - Mark bullet dead; remove from bullets set
   
5. Client receives:
   - KillPacket if headshot/kill
   - SyncedParticle for impact effect (decal, spark, blood)
   - UpdatePacket with victim health/armor state
```

### Projectile Lifecycle

```
1. Player throws grenade (ThrowableItem.use())
   └─> game.addProjectile({ owner, position, velocity, fuseTime })
   
2. Projectile created with:
   - hitbox: CircleHitbox(radius)
   - owner: the thrower (player/NPC)
   - fuseTime: countdown in milliseconds (default: definition.fuseTime)
   - _velocity: initial horizontal velocity (m/s)
   - _velocityZ: vertical velocity (gravity)
   - _height: Z-axis position (0 = ground, > 0 = air)
   
3. Each game tick:
   - Update height: _height += _velocityZ * dt
   - Apply gravity: _velocityZ -= gravity * dt
   - Clamp height: [0, maxHeight]
   - Query obstacles below by height
   - Apply drag based on surface (air, water, ice, ground)
   
4. Collision detection:
   - Check line from _lastPosition to position
   - For each object: get hitbox.intersectsLine()
   - Sort by distance (closest first)
   - For obstacles/buildings: ignore if height < projectile height
   - On collision: resolve hitbox collision, bounce velocity
   - If impactDamage defined: damage object
   
5. Fuse countdown:
   - _fuseTime -= dt
   - When _fuseTime < 0: call _detonate()
   
6. Detonation:
   - Create Explosion at projectile position
   - Create synced particles (visual effect)
   - Create decal (scorch mark)
   - Mark projectile destroyed
```

### Explosion Mechanics

```
Explosion.explode() raycast algorithm:

1. Query all objects in 2× max radius circle (broad phase)
2. Calculate angular step: 360° / (radius_max / explosionRayDistance)
   └─> typical: 16 rays at roughly 12.5° intervals
   
3. For each angle from -180° to +180°:
   - Create ray from explosion center to radius.max distance
   
   4. For each nearby object:
      - Check if hitbox.intersectsLine(center, rayEnd)
      - Get intersection.point and squareDistance
      - Add to lineCollisions array
   
   5. Sort lineCollisions by distance (nearest first)
   6. For each collision (nearest → farthest):
      - Mark object as damaged (track by object.id)
      - Calculate distance from center
      - Apply damage with falloff:
        damage = baseDamage × (1 - distance / maxRadius)
      - For obstacles: multiply by obstacleMultiplier
      - For players: check perk modifiers (e.g., low profile)
      - Apply knockback: angle (center → object), magnitude falloff
      - If obstacle has collisions and is not stair:
        BREAK (stop raycasting further; wall shields behind)
   
7. Create shrapnel bullets:
   - For each i < shrapnelCount:
     game.addBullet(explosion, source, {
       position: center,
       rotation: randomRotation(),
       idString: explosionDef.ballistics.idString
     })
   
8. Create decal if defined (visual scorch mark)
```

## Bullet Configuration (BaseBulletDefinition)

Defined per bullet type in `common/src/definitions/bullets.ts`:

| Field | Type | Purpose |
|-------|------|---------|
| `damage` | number | Base damage per hit |
| `speed` | number | Distance units per second |
| `range` | number | Max distance before deletion |
| `rangeVariance` | number? | Random variance in range (±%) |
| `obstacleMultiplier` | number | Damage multiplier for obstacles/buildings |
| `allowRangeOverride` | boolean | Can GunItem override range? |
| `noCollision` | boolean | Ignore collision detection (skip bullets) |
| `noReflect` | boolean | Prevent reflection off mirrors/surfaces |
| `onHitExplosion` | ExplosionDefinition? | Explosion triggered on impact |
| `explodeOnImpact` | boolean | Trigger explosion when hitting reflective surface (vs. bounce) |
| `onHitProjectile` | ThrowableDefinition? | Projectile spawned on impact (e.g., sticky bomb) |
| `infection` | number? | Infection damage (for special game modes) |
| `tracer` | TracerDefinition? | Visual bullet trail (color, width, opacity, spin) |
| `trail` | TrailDefinition? | Particle effect trail behind bullet |

### Tracer Configuration Example
```typescript
tracer: {
  color: 0xFFFF00,           // Yellow tracer
  width: 0.5,                // Scale 0.5x default
  opacity: 0.8,
  spinSpeed: 360,            // Degrees per second
  zIndex: ZIndexes.Ground
}
```

## Collision Detection

### Raycast Algorithm (Bullets)

```typescript
// From Bullet.update(), line ~115
const lineRect = RectangleHitbox.fromLine(
  this.position,
  Vec.add(this.position, Vec.scale(this.velocity, this.game.dt))
);
const objects = grid.intersectsHitbox(lineRect, this.layer);

// For each collision from updateAndGetCollisions(dt, objects):
for (const collision of this.updateAndGetCollisions(dt, objects)) {
  const { point, normal } = collision.intersection;
  
  // Check if should reflect (vs. die)
  const reflected = (
    collision.reflected
    && this.reflectionCount < 3
    && !definition.noReflect
    && (definition.onHitExplosion === undefined || !definition.explodeOnImpact)
  );
  
  if (reflected) {
    // Nudge position to avoid re-collision
    const rotation = 2 * Math.atan2(normal.y, normal.x) - this.rotation;
    this.position = Vec.add(point, Vec(Math.sin(rotation), -Math.cos(rotation)));
    
    // Recurse: create new reflected bullet with reflectionCount++
    this.reflect(rotation);
  } else {
    this.position = point;
    this.dead = true;
    break;
  }
}
```

**Key constraints:**
- Bullets only collide on the layer they were fired from
- Reflection count limited to 3 (max 4 bounces)
- Obstacles with `noCollisions` flag are pierced (bullets pass through)
- Stairs are special: don't stop bullets, but handle interaction
- Dead shooters' bullets are deleted immediately

### Projectile Collision (Grenades)

```typescript
// From Projectile.update(), line ~117
const objects = this.game.grid.intersectsHitbox(
  RectangleHitbox.fromLine(this._lastPosition, this.position),
  this.layer
);

// For each collision:
// 1. Check height: if projectile.height < obstacle.height, skip
// 2. Get hitbox intersection if collision.isLine = true
// 3. Resolve collision: hitbox.resolveCollision(object.hitbox)
// 4. Bounce velocity: _reflect(collision.normal)
```

**Height-based collision:**
- Projectiles have 3D position: `{ x, y, height }`
- Obstacles have 2D hitbox + `height` field
- Projectile passes through if `projectile.height > obstacle.height`
- Example: grenade can arc over a table (table.height = 1) but not over a tree (tree.height = 10)

## Damage Model

### Bullet Damage Calculation

```typescript
// From Bullet.update(), line ~137
const damageMod = (this.modifiers?.damage ?? 1) / (this.reflectionCount + 1);

const damageAmount = (
  definition.damage × damageMod
  × (isObstacle ? definition.obstacleMultiplier : 1)
);
```

**Modifiers applied:**
- `modifiers.damage`: Perk/item bonus (e.g., hollow point +10%)
- `reflectionCount`: Each bounce halves damage (1st bounce = 50%, 2nd = 25%, etc.)
- `definition.obstacleMultiplier`: Obstacles take more damage (e.g., 2× vs. players)

### Damage Type Interactions

**Teammate Healing (Team Mode):**
```typescript
if (
  definition.teammateHeal
  && this.game.isTeamMode
  && object.teamID === this.shooter.teamID
) {
  damageAmount = -definition.teammateHeal;  // Negative = heal
}
```

**Perk Modifiers:**
- `LowProfile`: Explosion damage reduced by perk's `explosionMod`
- `HollowPoints`: Highlighted indicator on hit player
- `PrecisionRecycling`: Refund ammo on headshot or miss

### Explosion Damage Falloff

```typescript
// From Explosion.explode(), line ~63
const dist = Math.sqrt(collision.squareDistance);

if (isObstacle) {
  damageAmount *= definition.obstacleMultiplier;
}
if (isPlayer) {
  damageAmount *= object.mapPerkOrDefault(PerkIds.LowProfile, 1);
}

// Distance falloff: linear from min radius to max radius
if (dist > min) {
  damageAmount *= (max - dist) / (max - min);  // 1.0 at min, 0.0 at max
}
```

**Example:**
- Explosion: `radius = { min: 8, max: 25 }`, `damage = 130`
- At distance 8: damage = 130
- At distance 16.5 (midpoint): damage = 130 × 0.5 = 65
- At distance 25: damage = 0

## Knockback

### Explosion Knockback

```typescript
// From Explosion.explode(), knockback applied to Loot/Projectile:
const multiplier = isProjectile ? 0.002 : 0.01;
object.push(
  Angle.betweenPoints(object.position, this.position),  // Push away
  (max - dist) * multiplier  // Magnitude scales with distance
);
```

**Direction:** Away from explosion center
**Magnitude:** Scales linearly with distance to explosion

### Bullet Knockback

Bullets don't apply knockback directly; only explosions do.

## Synchronization

### Network Serialization

Bullets and projectiles are **server-authoritative only** — clients do not simulate them:

| Event | Sent to Client | Via Packet Type |
|-------|---|---|
| Bullet impact | Kill/damage event | `KillPacket` |
| Impact visual | Particle effect | `SyncedParticle` |
| Projectile detonation | Explosion particles | `SyncedParticle` + `Decal` |
| Affected player update | Health/armor state | `UpdatePacket` (partial) |

### Update Packet (Partial Dirty)

When a projectile or bullet affects a player, the server marks the player as `partialDirty`:
```typescript
object.setPartialDirty();  // Flag for next UpdatePacket
```

This sends only the changed fields (health, armor, statusEffect) without full reserialization.

### Decal Synchronization

Explosions create **Decal** objects (server-side registered) that are automatically serialized to clients:
```typescript
const decal = new Decal(
  this.game,
  definition.decal,  // Decal definition (e.g., scorch mark)
  this.position,
  randomRotation(),
  this.layer
);
this.game.grid.addObject(decal);  // Synced to UpdatePacket
```

Decals fade out after `decalFadeTime` (if defined).

## Dependencies

### Depends on:

- **[Core Math & Physics](../core-math-physics/)** — Ray casting (`hitbox.intersectsLine()`), collision resolution (`hitbox.resolveCollision()`), vector operations (`Vec.add()`, `Angle.betweenPoints()`)
- **[Spatial Grid](../spatial-grid/)** — Object queries (`grid.intersectsHitbox()`, `grid.updateObject()`)
- **[Game Objects (Server)](../game-objects-server/)** — Damage handlers (`object.damage()`), pushback (`object.push()`), object lifecycle
- **[Object Definitions](../object-definitions/)** — `BulletDefinition`, `ThrowableDefinition`, `ExplosionDefinition` schemas and registries
- **[Game Loop](../game-loop/)** — Tick cycle integration (`game.addBullet()`, `game.addExplosion()`, `bullet.update()` called per tick)
- **[Inventory](../inventory/)** — `GunItem.definition` and `ThrowableItem` usage

### Depended on by:

- **[Game Loop](../game-loop/)** — Calls `bullet.update()`, `projectile.update()`, `explosion.explode()` every tick
- **[Game Objects (Server)](../game-objects-server/)** — Player/obstacle damage and knockback
- **[Game Objects (Client)](../game-objects-client/)** — Receives `SyncedParticle` and `Decal` for rendering
- **[Inventory](../inventory/)** — Weapon firing (`gun.use()` calls `game.addBullet()`)

## Known Issues & Gotchas

### 1. **Same-Tick Hits on Game Creation**
- **Symptom:** Bullet created and hits obstacle/player on the same tick
- **Root Cause:** Collision detection runs in same tick as creation; no grace period for initial position
- **Mitigation:** Game logic avoids placing bullets inside obstacles; `lastPosition` is initialized correctly

### 2. **Raycasting Cost is O(n)**
- **Symptom:** High CPU usage with many grenades (each projectile queries all objects per tick)
- **Current Scale:** 40 projectiles × ~100 objects = 4000 queries per tick (at 25ms = ~160k/sec)
- **Potential Optimization:** Spatial hashing of projectiles, or batching raycasts

### 3. **Reflection Count Unbounded Before v3**
- **Fixed in current code:** `reflectionCount < 3` limit (max 4 bounces)
- **Risk if changed:** Infinite bounces = infinite damage scaling

### 4. **Knockback Applied Instantly**
- **Issue:** Knockback velocity added directly in same tick as damage; can cause velocity spikes
- **Mitigation:** Knockback magnitude is clamped via `object.velocity` max cap (per player definition)

### 5. **Explosion Damage Can Stack in Same Tick**
- **Symptom:** Multiple bullets hit same object in one tick; all damage applies (no reduction)
- **Mitigation:** Possible in edge cases (shotgun burst), but expected behavior

### 6. **Hitscan Can Tunnel Through Walls on High Latency**
- **Symptom:** Client fires at target; network lag causes local position ahead of server; server raycast misses
- **Current Code:** No client-side prediction for bullet raycasts (server-authoritative)
- **Workaround:** Server-side lag compensation not implemented; relies on low latency

### 7. **Penetration is All-or-Nothing**
- **Code:** `if (isObstacle && object.definition.noCollisions) continue;` (pierce) else `break;` (stop)
- **Issue:** No per-wall degradation; bullet either pierces everything or stops on first hit
- **Potential Fix:** Track penetration points and reduce damage per pierce

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Physics tick integration, collision system overview
- [Data Model](../../datamodel.md) — DamageRecord, GameObject hierarchy

### Tier 2 — Subsystems
- [Game Loop](../game-loop/) — Bullet/projectile/explosion update integration; tick cycle (25ms, 40 TPS)
- [Game Objects (Server)](../game-objects-server/) — `GameObject.damage()` handler, knockback, health/armor
- [Core Math & Physics](../core-math-physics/) — Raycasting, angle, distance calculations
- [Spatial Grid](../spatial-grid/) — `grid.intersectsHitbox()` for collision queries
- [Object Definitions](../object-definitions/) — Registry lookups for bullets, explosions, throwables
- [Inventory](../inventory/) — `GunItem.use()`, `ThrowableItem.use()` firing mechanics

### Tier 3
- *No Tier 3 modules documented yet; see Tier 2 subsystems for implementation details*
